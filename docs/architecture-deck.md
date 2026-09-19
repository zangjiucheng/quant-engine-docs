---
title: quant-engine · architecture
sub_title: A C++20 backtest engine that grew a live trading daemon
author: architecture walkthrough · v0.4.1
theme:
  name: catppuccin-macchiato
---

# What this is

> **[▶ Open as slides](deck-en.html)** — arrow keys to navigate.
> This page is the same deck as a readable document; the slide markers
> below are presenterm directives and render as nothing here.
> The source runs in a terminal: `presenterm docs/architecture-deck.md`

A **C++20 quantitative trading system** in one repository.

- a zero-allocation backtest engine
- a `.qe` configuration language with its own evaluator
- a live IBKR trading daemon, running a paper account
- a terminal-style native dashboard — **the only shipped binary**

<!-- pause -->

```
C++      370 files        13,101 graph nodes
Bash      23              46,292 edges
Python     5              ~220,000 lines
TypeScript 1              3,244 tests
```

<!-- pause -->

The interesting part is not the size. It is **what the code refuses
to do**, and how much of that refusal is structural rather than
advisory.

<!-- end_slide -->

# The one idea, up front

Six of the last twelve production defects had the same shape:

> a health signal reading **all-clear** over a channel
> it was **not measuring**

<!-- pause -->

`feed_healthy` green through a session with zero bars.
`fill_integrity` polled on the socket that places no orders.
`unbacked_sells` counted on an empty book.
`broker_connected` as one boolean over two sockets.

<!-- pause -->

**Three of them shipped _as monitoring_.**

<!-- pause -->

That produced a rule, now in `CLAUDE.md`:

> A health signal ships with a test that drives it **RED**,
> or it does not ship.

<!-- end_slide -->

# Map

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Offline**

```
  .qe config
      |
   evaluator      <- cross-sectional
      |              prepass
  MultiEngine     <- the hot loop
      |
  Portfolio
      |
  metrics / IC / walk-forward
      |
  results.json + report.html
```

<!-- column: 1 -->

**Live**

```
  qe_daemon
      |
  LiveEngine
      |
  OrderScheduler   <- staged slices
      |
  [ safety stack ]
      |
  IbkrOrderRouter
      |
   TWS  (2 sockets)
```

<!-- reset_layout -->

The dashboard attaches to the daemon **read-only, over the network**.

<!-- end_slide -->

# Part I — The engine

<!-- end_slide -->

## Data layout: struct-of-arrays, enforced

```cpp
// CORRECT
struct BarSeries {
    std::vector<int64_t> timestamps;   // ns since epoch
    std::vector<double>  open, high, low, close, volume;
};

// WRONG — CLAUDE.md prints this under
//         "do not introduce this"
struct Bar { int64_t ts; double o,h,l,c,v; };
std::vector<Bar> bars;
```

<!-- pause -->

Three reasons, not one: column scans hit cache **linearly**; the
loop auto-vectorizes on AVX2/NEON; and indicators take one column,
not a stride.

<!-- pause -->

Multi-symbol is **symbol-major** `vector<BarSeries>`, indexed by
`size_t`. `unordered_map<symbol, BarSeries>` is rejected by name —
*"hash lookup per bar per symbol is forbidden in the hot loop."*

<!-- end_slide -->

## Dispatch: CRTP, and one place virtuals are allowed

```cpp
template <typename Derived>
class IStrategy {
    void on_bar(std::size_t i, const BarView& b, Portfolio& pf) {
        static_cast<Derived*>(this)->on_bar_impl(i, b, pf);
    }
};
```

No vtable per bar; the call fully inlines.

<!-- pause -->

Virtuals are permitted in **exactly one file** — `factory.hpp`, the
JSON/`.qe` config boundary, which says so in capitals:

> `VIRTUAL FUNCTIONS ARE ALLOWED HERE — and only here`

<!-- pause -->

`BarView` is a **by-value 48-byte copy**, not pointers into the columns:
pointers would tie the view's lifetime to the series and force
pointer-chasing across 5+ allocations per field access.

<!-- end_slide -->

## The loop: look-ahead is unrepresentable

The engine is one templated `for`. Its **statement order** is the safety
property:

```
bar i-1   strategy sees bar i-1
          submit_buy() -> sets pending_qty. Nothing executes.
                                    |
                                    | intent crosses the boundary
                                    v
bar i     (1) fill pending_qty at bar i's OPEN
          (2) mark_to_market at bar i's CLOSE
          (3) NOW call strategy.on_bar(i, ...)
                                    |
                                    | can only submit into i+1
                                    v
bar i+1
```

<!-- pause -->

Look-ahead is not prevented by a check that could be bypassed.

**Bar `i+1` has not been constructed when the strategy runs.**
There is no object to peek at.

<!-- end_slide -->

## Zero allocations, and the test that proves it

Inside `Engine::run()`: no `new`, no `push_back` without `reserve`,
no `std::string`, no map lookups, no exceptions.

<!-- pause -->

Verified by a **counting allocator**, not by inspection —
`tests/alloc_counter.{hpp,cpp}` + 15 tests.

<!-- pause -->

| target | measured | margin |
| --- | --- | --- |
| 10y daily < 10 ms | **0.022 ms** | 454x |
| 1y minute < 100 ms | **0.916 ms** | 109x |
| 10k-cell sweep < 30 s | **0.554 s** | 54x |
| CSV > 500 MB/s | **514 MB/s** | pass |

Sweeps parallelize at the **configuration** level — never inside a
run. Each cell gets its own Portfolio, Engine and buffers.

<!-- end_slide -->

# Part II — The `.qe` language

<!-- end_slide -->

## One AST, two languages

```
backtest(...) sweep(...) walk_forward(...) -> qe_run
research(...)                              -> qe_factor
```

<!-- pause -->

One lexer and one AST serve **two** layers:

| | config layer | signal layer |
| --- | --- | --- |
| when | once, at load | every bar |
| yields | typed `QeValue` | a double |
| builtins | 26 | 25 |
| allocates | freely | **never** |

The signal expression is **sliced out of the same tree** and re-bound to
numeric ids with pre-allocated state slots. A `let fast = 10` inlines
into the signal tree as a constant, for free.

<!-- pause -->

Cost, stated honestly: `BM_Dsl_MaCross` **0.166 ms** against
`BM_Hardcoded_MaCross` **0.022 ms** on the same 2,500 bars. The DSL is
7.5x the hand-written strategy — and still 60x under the target.

<!-- end_slide -->

## The rule that looks like a bug

**`and` / `or` do not short-circuit. Both branches always run.**

<!-- pause -->

Because every stateful indicator must be fed **exactly once per bar**.
If a surrounding `and` skipped its right operand on a bar where the left
was false, that `sma` would miss a sample and desync from the price
series — permanently, and silently.

<!-- pause -->

Same reason, same file: every call argument is evaluated even when
unused, window literals walked with `(void)eval_node(...)`.

> an arg that is sometimes walked and sometimes not
> is how a stateful indicator desyncs

<!-- pause -->

Correctness for a *streaming* evaluator is not the same property as
correctness for an expression evaluator. The obvious optimization is
the bug.

<!-- end_slide -->

## Seven primitives, one window

Across a universe on one bar, **seven** cross-sectional primitives must
address the *same* block of scored symbols:

```
   [ -- non-finite -- | ------ SCORED ------ | -- non-finite -- ]
                      ^                      ^
                 xs_finite_lo_[s]     + xs_finite_n_[s]
```

<!-- pause -->

Computed **once per site**, by the prepass. It used to be recounted
inside `fill_xs_norm` and nowhere else — *"which is precisely why
`is_top`, a selection-only site that never calls this function, was left
comparing against a different window."*

<!-- pause -->

The consequence: one site answering **two different questions** about
the same member. `rank(x) == n-1` and `is_top(x, 1)` named different
symbols on the same bar; an infinity carried a percentile while its
z-score was NaN.

"Unusable" means `!isfinite`, not `isnan` — **an infinity is not a
ranking**, but it passed an `isnan` guard and got selected as the single
best name in the universe.

<!-- end_slide -->

## Where the complexity actually lives

Cyclomatic complexity across the whole repository:

```
call_builtin      src/dsl/config_eval.cpp        513
eval              src/dsl/config_eval.cpp        104
process_input     apps/dashboard/vim_editor.cpp   76
tick              src/live/order_scheduler.cpp    59
clone_for_signal  src/dsl/config_eval.cpp         49
```

<!-- pause -->

One function is 5x the next. That is the DSL builtin **dispatch** — a
switch, not logic, and the shape you want complexity to take: wide and
flat, in one place, rather than smeared thin across the codebase.

<!-- pause -->

Compare the hot path: `BM_XsPrepass_30sym_IsTop10` runs in **1.62 µs**.

<!-- end_slide -->

# Parity

*the spine — what makes the backtest worth anything*

<!-- end_slide -->

## A gate is worthless if the two disagree

> The backtest is what **gates** a strategy;
> the daemon is what **trades** it.

`docs/safety-and-limitations.md` carries a 13-row parity matrix, and
says every row was **read from source on the current build, not from
the docs**.

<!-- pause -->

The row that makes the rest possible:

| dimension | backtest | live | ? |
| --- | --- | --- | --- |
| factor evaluator | `qe::dsl::Evaluator` | same class + AST | YES |
| xs rank algorithm | `cross_sectional_prepass` | same function | YES |

> `on_bar` is evaluated by the **SAME** `qe::dsl::Evaluator` the
> backtest engine uses. Live mode is just *"feed the same Evaluator
> with bars as they arrive instead of a pre-known series."*

<!-- pause -->

Not "ported to". Not "kept in sync with". **The same code.**

<!-- end_slide -->

## And the rows that do not agree

| dimension | backtest | live | ? |
| --- | --- | --- | --- |
| `whole_shares = true` | **ignored** | applied | NO |
| `compound = true` | applied | no live path | NO |

<!-- pause -->

`whole_shares` floors each live order to an integer share count.
The backtest does not model it. `grep whole_shares` across
`include/qe/backtest`, `include/qe/strategy` and `src/backtest` returns
**nothing** — it exists only in `src/live/live_engine.cpp`.

Measured consequence: **~14% of capital left idle** by integer
rounding.

<!-- pause -->

And it was invisible to the gate **because the gate is the backtest**.

<!-- pause -->

That is why the daemon has a startup gate that **refuses to run**
(exit 4) when live cannot reproduce backtest sizing:

> Never degrade quietly. A silent sizing divergence cost the 2026-06
> paper deployment six weeks — it submitted orders, looked alive in
> every log, and **bought one share per leg**.

<!-- end_slide -->

# Part III — The live path

*the centre of gravity*

<!-- end_slide -->

## Every layer exists because a green light lied

Two TCP sockets to the same gateway — market data on `client_id`,
orders and positions on `client_id + 1`.

Not style: IB Gateway **forbids** two connections sharing a client id,
and it interleaves frames across open sockets, so an idle peer's
handshake can be delayed indefinitely.

<!-- pause -->

```
  intent
    |
  ReconcileGatedRouter   <- outermost, can only REFUSE
    |
  PreTradeRisk           <- exposure caps, 3 pause sources
    |
  SafeBroker             <- per-order limits, kill switch
    |
  IbkrOrderRouter        <- 10 s ack deadline
    |
  TWS
```

Each layer answers a **different** question against a
**different** book.

<!-- end_slide -->

## The reconcile gate — five rules, each load-bearing

On **2026-07-31** the daemon booked ten fills at 09:30:31–33. Its own
reconcilers contradicted every one within ninety seconds, all reading
`broker=0`.

**It kept trading against a book that did not exist.**
Next morning: account flat, no working orders, equity unchanged.

<!-- pause -->

> A detector whose only output is a log line
> is a detector nobody acts on.

<!-- pause -->

1. **Additional** gate — can only turn Accepted into a refusal
2. **Nothing is auto-adopted** — it never writes the broker's number
3. Cleared by **one** thing: an operator ack naming a decision
4. Watermarked per `(kind, symbol)` — no persisted "ignore forever"
5. **Starts HELD** — fail-closed; a skipped evaluation costs a daemon
   that will not trade, never one that trades unreconciled

<!-- end_slide -->

## The cost, stated rather than buried

The gate's own header has a section with that title:

> A block gates the strategy's **exits** as well as its entries.
> There is no way to gate one and not the other honestly: an exit is
> sized against a position the block exists because we cannot describe.
>
> A daemon holding real positions under a block is
> **not managing them**.

<!-- pause -->

And what deliberately does **not** block:

- an **inconclusive** reconcile — "we could not see the venue" is not
  evidence of drift, and blocking on it would hand IB Gateway's nightly
  restart a veto over the next session
- **cancels**, ever — they only reduce exposure

<!-- pause -->

A safety document that only lists strengths is marketing.

<!-- end_slide -->

## A breaker that could not trip

The daily-loss breaker used to measure the **account**:

```
account net liquidation    ~$1,000,000   (paper)
threshold                      $15,000
max possible strategy loss      ~$8,333   <- gross
```

<!-- pause -->

The strategy could lose **everything it had** and never move the number
being watched. The breaker was mathematically incapable of tripping.

<!-- pause -->

Now both layers measure the **strategy's** own book against the
strategy's own anchor. The strategy's book and the account are
**different objects** — the account also holds shares placed by hand,
declared via `qe_daemon baseline adopt`.

Raising a cap is not a fix for a scope error.

<!-- end_slide -->

## Selected constants

```
order ack deadline            10 s
snapshot queries              15 s
disconnect watchdog           1 Hz; soft-pause at 3, kill at 30
feed silence limit            600 s  (~10x worst observed gap)
fill reconciler               60 s poll, 2 passes to confirm
                              1e-6 share tolerance
equity poller                 30 s  (~780 polls/session)
journal rotation              64 MiB (~267 KB / 3 weeks observed)
TWS PLACE_ORDER               pinned to v151, static_assert
```

<!-- pause -->

One `submit` was observed blocking for **177 s** on 2026-06-16.
Ten legs x 30 s = exactly the 300 s window the scheduler tolerates.

Numbers here are sized against **observed** worst cases, not guesses.

<!-- end_slide -->

## Two failures that logged nothing

**The warmup constant.** The daemon asked IBKR for `"60 D", "1 day"` —
about 41 bars — for every symbol, a literal unchanged since it was
written.

> A config needing more did not fail; it started, logged nothing
> unusual, and **traded on NaN factor values**.

And NaN sorts last in the cross-sectional prepass — so a long-only
`is_top` quietly picked whichever handful of symbols happened to be
warm. Now sized from the config, and it **refuses to start if short**.

<!-- pause -->

**The one-way kill switch.** It trips after ~30 s of disconnect, is
deliberately one-way, and the daemon **does not exit** — it stays alive,
permanently paused.

> Process supervision (launchd `KeepAlive`, systemd `Restart=`)
> does **NOT** help, because nothing died.

2026-06-19: disconnect kill at 19:58 ET. The daemon sat paused for
**two and a half weeks**.

<!-- pause -->

The supervisor that clears it is fenced four ways: paper accounts only,
`kill_reason == ibkr_disconnect` exactly, venue reachable *before*
restart, and at most N restarts per rolling 24 h.

Auto-clearing a safety control is allowed to exist — but it has to
argue for itself.

<!-- end_slide -->

# Part IV — Analytics

<!-- end_slide -->

## Measured, or absent — never zero

> Every number this layer publishes is either measured
> or spelled as absent. Never as zero.

<!-- pause -->

- `win_rate` serializes as **`null`**, not `0.0`, when round trips
  do not pair across a multi-symbol book
- `provenance.not_computed` **names** what was skipped and why
- an optimizer **refuses** a metric it cannot rank rather than scoring
  the cell zero — a measured cell always outranks an unmeasured one
- `positions_checked` distinguishes *"the books agree"* from
  *"the books were never compared"*

<!-- pause -->

The principle, stated once and applied everywhere:

> **An absent expectation is not an expectation of zero.**

Rendering silence as `0` turned every position the account legitimately
held into a drift row.

<!-- end_slide -->

# Part V — The dashboard

<!-- end_slide -->

## Read-only, enforced by the daemon

An 8-screen immediate-mode GUI: GLFW + Dear ImGui + ImPlot.
Live quotes, candlesticks, a live view of any external `results.json`,
an mtime-watched positions file — and a **vim-style editor**
inside it.

<!-- pause -->

The architectural decision is not the GUI. It is this:

> The dashboard attaches to a remote daemon read-only, and the
> read-only property is enforced by **`ReadOnlyControlHandler`
> in the daemon** — not by dashboard convention.

<!-- pause -->

A client cannot become trusted by lying about what it is.

<!-- pause -->

`qe_dashboard` is also split so the **test suite does not link the GUI
half** — the deployment host has no graphics stack. That split is
load-bearing: the headless nix shell deliberately omits X11, which is
what makes a regression in it visible.

<!-- end_slide -->

# Part VI — Discipline

<!-- end_slide -->

## There is no CI. Deliberately.

So every gate that would live in a pipeline was pushed somewhere a human
actually meets it:

- a **preset description** that argues with you
  (`dist` explains why `-march=native` cannot ship: it encodes the
  *build* machine's CPU, so a release cut on an M4 SIGILLs on every M1)
- a **publish script** that gates the bundle: every Mach-O must link
  only `/usr/lib` and `/System`, be arm64, and carry the exact `minos`
  the download page advertises
- a **weekly open-check** with a real exit-code vocabulary:
  `4` no daemon · `6` no target configured · `7` target unreachable
- a **docs pin** in `requirements-docs.txt`, because a bare `mkdocs`
  is whatever the machine happens to provide

<!-- end_slide -->

## Three ways a "red test" proves nothing

All three have shipped here.

<!-- pause -->

**1. The red is a build failure.**
ctest then runs a **stale binary** and reports PASS.
Confirm the removed-fix build exits 0.

<!-- pause -->

**2. The red comes from breaking the test, not removing the fix.**
That proves only that the test can fail.

<!-- pause -->

**3. The red is a unit test of a component that was never broken.**
`OrderLog` was constructed, passed to `BrokerSession::wrap`, stored, and
read back as truth — while nothing between them ever called `append()`.
A 0-byte file for the whole deployment.

Tests attaching the log directly to `SafeBroker` stayed **green** with
the production wiring deleted. Only a composition-level test caught it.

<!-- pause -->

> Delete the production line you think fixes the bug,
> then see which test notices.

<!-- end_slide -->

## Benchmarks are claims about machine state

Every file in `benchmarks/results/` carries the load average it was
taken under, and says what was running.

<!-- pause -->

`BM_QeSweep` read **+21%** at v0.4.1 and was **not** recorded as a
regression:

- the three untouched control benches moved **+2 to +6%** the same way
  — the machine, not the code
- 0.554 s sits **inside the baseline binary's own spread**
  (0.457 / 0.478 / 0.614 / 0.715 across four runs)
- re-measured at load 4.2 it returned 0.71–0.76 s; at load 2.6, 0.554 s

<!-- pause -->

> A snapshot taken under load and filed without that note
> is worse than no snapshot — it becomes the baseline someone
> else measures a phantom regression against.

<!-- end_slide -->

## The project documents its own violations

`CLAUDE.md` says: one EPIC per branch, operator approves, operator
merges. `EPICS/README.md` then says, under a heading that begins
**"a workflow deviation, on the record"**:

> **84, 85, 86, 87 and 88 were completed directly on `main`.**
> No `epic/NN-...` branch, no pre-merge review gate.
>
> The acceptance is real and each file records what it rests on.
> The merge discipline was not followed. **Both facts belong here.**

<!-- pause -->

Likewise EPIC-83's 15 residuals, re-verified against `main` on
2026-07-30: **none is closed.** Each is tagged fail-closed or fail-open,
R1 still blocks redeploy, and two acceptance lines are recorded as
having shipped **without evidence**.

<!-- pause -->

A repository that only records its successes is a repository whose
records you cannot use.

<!-- end_slide -->

# What to take away

<!-- pause -->

**1. Make invariants structural, not advisory.**
Look-ahead bias is impossible because bar `i+1` does not exist yet —
not because a comment asks you not to peek.

<!-- pause -->

**2. "I cannot tell" must never render as "all clear".**
Six defects in one family. Absence of a measurement and absence of a
finding are different sentences.

<!-- pause -->

**3. State the cost of your own safety mechanism.**
The reconcile gate documents that it blocks exits too, and that a daemon
under a block is not managing its positions.

<!-- pause -->

**4. A test is only real if you have watched it fail** —
against the removed *fix*, at the seam where the defect lives.

<!-- end_slide -->

# Questions

```
  repo      ~220k lines · 3,244 tests · no CI
  binary    QE Dashboard.app  (arm64, not notarized)
  daemon    IBKR paper, one strategy, 30 symbols
  docs      mkdocs -> GitHub Pages
```

`CLAUDE.md` is the conventions document.
`benchmarks/README.md` is the measurement protocol.
`EPICS/` is the work history.
