# Python

There are **two** ways Python touches this project, and they are not
the same thing. One needs nothing installed; the other compiles the
engine.

| | What it is | Needs |
|---|---|---|
| **Daemon client** | Read and reconfigure a running `qe_daemon` over its control socket | A Python interpreter. Nothing else. |
| **quant-engine-py** | nanobind bindings — run backtests and analytics in-process | A C++ toolchain, vcpkg, and a compile of the engine |

If you want to watch or steer a live daemon, you want the first. If
you want to drive the backtest engine from a notebook, you want the
second.

## The boundary, for both

> **Python may start a run and read its results. Python may not be
> called from inside the engine's per-bar loop.**

This is a correctness property, not a style preference. The hot loop
is allocation-free with no virtual dispatch — held by a counting
allocator in the engine's own test suite — and the performance
numbers in [Architecture](architecture.md) assume it. A per-bar Python
callback would take the GIL once per bar and allocate on every one,
voiding those numbers while the allocation test still passed, because
the allocations would happen on the Python side where it is not
looking.

So there is no way to write a strategy in Python here. Strategies come
from a `.qe` file. That is enforced by a test, not by convention.

---

## 1. The daemon client

Ships **in the engine repo** at `python/qe_daemon/`. Standard library
only — it links nothing and builds nothing, so it runs from a machine
with no engine on it. That is the same property that lets the
dashboard attach to a daemon on another host.

```python
import sys; sys.path.insert(0, "python")
from qe_daemon import DaemonClient, DaemonRefused

c = DaemonClient("/run/user/1002/qe_daemon.sock")
print(c.status()["slices_pending"])
print(c.reload_check(config="live_config.qe")["changes"])
```

Read verbs hang off the client; mutating verbs hang off `c.control`
— `c.control.pause()`, `c.control.reload_apply(...)`. That mirrors the
daemon's own `book` / `book_rebuild` split, so a mutation is visible
at the call site rather than hidden in a keyword argument.

**The split is naming, not enforcement.** The daemon allow-lists the
read verbs by name and refuses everything else, per connection.
`c.control.kill()` against a read-only socket raises because the
daemon said no — not because the client declined to send it. That is
the property worth having, and it holds only because the policy is not
duplicated in the client.

Errors are typed and **nothing returns a default**: `DaemonRefused`
(the daemon answered no, carrying its own message), `DaemonUnreachable`
(socket), `DaemonProtocolError` (the reply did not match the
envelope). A verb your daemon predates raises `DaemonRefused` naming
it, which is how you feature-detect.

That last point is not fussiness. A previous probe read a key at the
wrong nesting level, got a default, and reported an empty account for
six hours against a daemon holding ten positions. The client
subscripts everywhere so a shape change raises instead of reporting a
plausible nothing.

See the [daemon runbook](live-trading-runbook.md#driving-the-daemon-from-python)
for the operational context and for re-pinning the wire format.

---

## 2. quant-engine-py — the bindings

A **separate repository**, deliberately: the engine stays pure C++,
its dependency manifest is untouched, and the headless deployment host
never grows a Python build chain.

It is **private**, like the engine itself, and **not on PyPI**.
Publishing a wheel that links a private engine needs a deliberate
answer rather than a default; that answer is "later, once the surface
has settled". For now it is a local editable install.

### What is bound

```python
import qe

# bars — struct-of-arrays in, read-only numpy views out
s  = qe.BarSeries.from_arrays(timestamps=..., open=..., high=...,
                              low=..., close=..., volume=...)
mb = qe.MultiBarSeries.from_series(symbols=["AAA", "BBB"], series=[s, s2])
mb.align_intersection()
mb.bars[mb.symbol_index("AAA")].close     # float64 view, no copy

# a .qe, evaluated and run
cfg  = qe.load_qe("strategy.qe")
res  = qe.run_backtest(cfg, qe.load_bars(cfg))

# analytics
qe.ic_analysis(...)       qe.rolling_ic_analysis(...)
qe.long_short_backtest(...)
qe.sharpe(...)  qe.sortino(...)  qe.max_drawdown(...)
qe.cagr(...)    qe.total_return(...)  qe.kahan_sum(...)
```

Not bound yet: sweeps, walk-forward, research configs, and individual
trades as objects.

### Three things that will surprise you if nobody says them

**Bar columns are views over the engine's own buffers**, not copies,
and they are **read-only**. `BarSeries` is parallel `std::vector<double>`
by architectural rule, which is already numpy's layout — that is the
one place this design is pleasant to bind. Read-only because the
engine's buffers are not Python's to write through, and a per-bar
mutation surface is the boundary above in a different costume.

**`run_backtest` returns numbers and no provenance.** A run handed
caller-supplied bars must not publish an artifact naming a data file
it never read; nothing downstream could detect that forgery. No
library entry point produces provenance at all — the only thing that
does is the `qe_run` CLI, whose path loads from the config it records.
So if you need a reproducible artifact, run the `.qe` through
`qe_run`. A test asserts both halves of this.

**Drawdown has two sign conventions**, and this is the engine's, not
the binding's: `qe.max_drawdown` returns a **positive** fraction
(`0.25` = gave back a quarter), while `LongShortResult.max_dd` is the
most **negative** excursion (`-0.202`). Compare like with like.

### Version alignment, which is the real cost

Two repos, no CI, and a C++ API with no stability guarantee. The
failure mode is a binding compiled against one engine and read as
though it spoke for another: it imports, it runs, and it returns
subtly different numbers.

Three things hold the line:

1. The engine is a submodule **pinned at a tag**, never a branch. The
   consequence is deliberate: the bindings lag the engine by one
   release. A commit landing on the engine's `main` today is not
   reachable from Python until the next tag.
2. The extension carries the version and commit it was compiled from
   — the version read from the engine's own generated constant, not a
   literal the bindings maintain. At import, if the submodule source
   is present, its current commit is compared and a mismatch
   **raises**. Not warns: the whole point is that the mismatch is
   invisible in the results.
3. A parity test runs one `.qe` through both `qe_run` and the
   bindings and requires the equity curve to match point for point,
   with `qe_run` built from the same pinned tag so a version
   difference cannot be mistaken for a binding bug.
