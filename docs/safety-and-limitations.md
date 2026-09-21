# Safety and known limitations

Current model constraints, backtest/live differences and operational limits.
Limitations stay here until fixed; version-specific migrations and dated
evidence live in [Migration and incident history](release-history.md).

Before deployment, also read the [safety model](live-trading-safety.md)
and [runbook](live-trading-runbook.md).

## Semantic traps

A semantic trap is a function whose name suggests one measurement and
whose arithmetic performs a different one. The engine currently detects
exactly one of them, which should tell you how much of this category is
still unmapped.

### `rolling_vol` is not volatility

`rolling_vol(x, n)` is the sample standard deviation of `x` over `n`
bars. Applied to a price — `rolling_vol(close, 20)` — that is the
standard deviation of the **dollar price level**, in dollars. It is not
return volatility, and it does not behave like it.

`realized_vol(x, n)` is the standard deviation of one-bar returns. It
is bit-identical to `rolling_vol(lag_return(x, 1), n)`.

Two synthetic 60-bar series make the difference concrete. TREND climbs
100 → 200 on a smooth exponential ramp, so every one-bar return is
about 1.2 % and they are all nearly identical. CHOPPY alternates
98 / 102, a ~4 % move every single bar.

| | `rolling_vol(·, 20)` | `realized_vol(·, 20)` |
|---|---|---|
| TREND (smooth ramp) | **12.455** | 9.0e-17 |
| CHOPPY (4 % a bar) | **2.052** | **0.0411** |

A low-volatility factor is normally written `rolling_vol(close, 20) * -1.0`
— higher score, lower value. Read that column: it scores CHOPPY *above*
TREND. **It prefers the name a low-volatility strategy exists to
avoid.**

The second measurement is less contrived and worse. Three names priced
at $30 / $180 / $900, constructed to have *identical* return series:

| | `rolling_vol(close, 20)` | `realized_vol(close, 20)` |
|---|---|---|
| $30 name | 0.3078 | 0.020318 |
| $180 name | 1.8468 | 0.020318 |
| $900 name | 9.2338 | 0.020318 |

`realized_vol` returns the same number for all three, as it must.
`rolling_vol` returns values exactly proportional to the share price,
30× across the range. So on a universe of mixed-price names,
`-rolling_vol(close, n)` is, to a first approximation, a **low
share-price factor**. Measured independently on real data, it is ~0.8
rank-correlated with `close` itself and ~0.12 with true return
volatility.

None of this is a bug in `rolling_vol`. It computes what it computes,
correctly. The trap is that the name reads like an answer to a
different question.

### What the engine does about it

`rolling_vol`'s arithmetic is **frozen and will stay frozen** — a
deployment trades it, and silently redefining a function under a
running strategy is its own failure class. Instead:

| Boundary | Behaviour |
|---|---|
| Binder / config load | Non-fatal lint whenever `rolling_vol` is applied to a bare `close` / `open` / `high` / `low` |
| `qe_run`, `qe_factor`, sweeps, walk-forward | Warn and run. Reproducing what a legacy config actually did has to keep working |
| `qe_daemon validate` | **Exit 4.** Config is not deployable |
| `qe_daemon start` | **Exit 4**, before any socket to the venue is opened |
| `results.json` | The finding is recorded in `provenance.warnings` with a stable code |
| `report.html` | Amber banner above the KPI cards |
| `qe_run` console | Printed at the bottom of the summary block, on **stdout** |

The last three exist because the first four were not enough. A warning
in a log is only a diagnostic if somebody reads it; see
[six weeks on the wrong factor](release-history.md#2026-06-to-2026-07-31-six-weeks-on-the-wrong-factor).

Two ways past the live refusal:

```qe
rolling_vol(price_level(close), 20)   // deliberate: no lint, no refusal
```

`price_level(x)` returns `x` and allocates no state. The arithmetic is
unchanged to the last bit, so a strategy that genuinely trades dollar
dispersion can adopt it with **no change to its results**. This is the
good option.

The other is `--allow-price-level-vol`, which starts the daemon and
prints the entire refusal at ERROR on every startup. It buys you a
running daemon; it does not buy you a quiet log.

### What is *not* linted, and why

- **`rolling_zscore(close, n)`** — `(close - mean) / std` is dollars
  over dollars, hence dimensionless. A perfectly reasonable
  standardized mean-reversion factor. It does not carry this defect.
- **Anything else.** The live gate is an **allow-list of blocking lint
  codes**, not "everything the binder can say". Adding a new diagnostic
  must not turn into an unannounced deploy gate for configs that were
  fine yesterday. The cost of that choice is that a future trap is not
  automatically fatal.

### The general case is open

One trap is detected. The engine has no general mechanism for "this
expression does not measure what its name implies", and no unit system
that would let it derive one. If you write a factor, check its scale
invariance yourself: multiply an input series by 10 and see whether the
factor moves.

---

## Backtest/live parity matrix

The backtest is what gates a strategy; the daemon is what trades it. A
gate is worthless if the two disagree. Every row below was read from
source on the current build, not from the docs.

| Dimension | Backtest | Live | Agree? |
|---|---|---|---|
| Factor evaluator | `qe::dsl::Evaluator` | same class, same bound AST | ✅ |
| Indicator state after warmup | full history | derived warmup window | ⚠️ [four classes](#indicator-state-after-warmup-four-classes) |
| Warmup history | full series from `from` | derived from the config; **refuses to start if short** | ✅ since 0.4.0 |
| Cross-sectional rank algorithm | `cross_sectional_prepass` | same function | ✅ |
| Cross-sectional batch trigger | every bar, all symbols present | quorum-or-timeout barrier; stale slots on timeout | ⚠️ by design |
| Entry / exit pre-pass | yes | yes | ✅ |
| Sizing pre-pass and warmup | yes | yes | ✅ since 0.4.0 |
| Inverse-vol allocator | pure function over pre-pass values | same function, same values | ✅ |
| Rebalance phase | `bar_index % n` | `flush_idx % n` | ✅ |
| `execution(whole_shares = true)` | **ignored** | applied | ❌ |
| `execution(compound = true)` | scales entries by `equity / initial` | **no equivalent** | ❌ |
| Resizing an open position | never | never | ✅ (and both are [wrong](#neither-runtime-resizes-an-open-position)) |
| Fills | modelled at the next open/close, plus fixed commission and slippage in bps | whatever the venue does | by definition ❌ |

### Indicator state after warmup: four classes

Measured against a frozen 600-bar × 3-symbol fixture, comparing a live
evaluator warmed to its minimum history against a backtest evaluator
fed the whole series. The first draft of that test asserted a clean
split into "exact" and "not exact"; two of its cases failed, and the
failures are the finding.

| Class | Indicators | Live vs backtest | Why |
|---|---|---|---|
| **Exact** | `lag`, `lag_return`, `rolling_max`, `rolling_min` | bit-identical | a ring of values and comparisons; nothing accumulates |
| **Exact after 0.4.0** | `rolling_vol`, `rolling_zscore`, `realized_vol` | bit-identical | the ring was summed in *physical slot order*, so the result depended on how many bars had been seen. Now summed oldest-first |
| **Drifting accumulator** | `sma` | ~1e-15 relative | `sum += x - buf[head]` is O(1) by design; rounding accumulates over all history. Not reorderable, and two-pass would cost ~200 flops/bar for `sma(200)`. **Stated, not fixed** |
| **IIR** | `ema`, `rsi` | decays as `(1-α)^k`, **never reaches zero** | an exponential filter never fully forgets its seed. Not pointwise-monotonic either: the seed moves with the window start, so the envelope decays, not each step |

Practical reading: a strategy built only from rows 1 and 2 has live
factor values bit-identical to its backtest's. A strategy using `ema`
or `rsi` does not, ever, at any warmup budget. That is a property of
the indicator, not a defect the engine can close, and the daemon says
so at startup with the residual spelled out.

The `sma` drift is ~1e-15 relative — it cannot flip a ranking except on
an exact tie. It is listed because "approximately equal" is a different
claim from "equal", and a page like this is where the difference
belongs.

### `whole_shares` is unmodelled in the backtest

`execution(whole_shares = true)` floors each live order to an integer
share count. Some venues and accounts reject fractional orders
outright, so it is not optional there.

**The backtest ignores the flag entirely.** It sizes off the literal
share count with no rounding. On a deployment that set it, the measured
consequence was **about 14 % of capital left idle** by integer
rounding — capital the backtest had fully deployed.

So a live book runs below its backtest by roughly that much, and no
gate saw it, because the gate is the backtest. If you set
`whole_shares`, treat every backtest return as an overstatement and
size positions so each leg comfortably clears one share.

### `compound = true` has no live equivalent

`execution(compound = ...)` defaults to **false**, and under that
default the two runtimes agree bar for bar.

Set it to `true` and only the backtest changes: entries scale by
`equity / initial_capital`. The live engine has no such path — it sizes
from the static per-leg capital budget. Read from the two sizing paths,
not measured against a broker: a config that sets `compound = true`
does not have live/backtest sizing parity.

The default has its own consequence, and it is easy to miss. With
`compound = false`, position sizes never grow as the account does. A
strategy that quintuples its equity is, by the end, taking the same
dollar risk it took on day one — **its risk per dollar of equity decays
across the whole run.** Measured on one 11-year study: daily
percent-return volatility fell from 1.455 % (2016) to 0.564 % (2025),
a 2.6× de-risking, while a buy-and-hold benchmark's stayed flat at
1.2–2.8 %.

If you compare a `compound = false` strategy's drawdown against
buy-and-hold, **you are comparing a book that shrank its risk against
one that did not.** That comparison is not like-for-like, and in the
study above the strategy's drawdown advantage did not survive being
corrected for it. Setting `compound = true` is not a fix either: it
interacts with the gross-exposure limitation below and, in that study,
produced a −112 % maximum drawdown.

#### The decay is now measured, and the default still does not flip

A later review asked whether `compound` should default to **true** instead, on
the grounds that the decay above is a defect rather than a setting.
It does not flip, for two reasons that are not preference:

- The daemon **refuses to start** on `execution(compound = true)`
  (`apps/daemon/sizing_parity.hpp`; also the ⛔ row in the
  [runbook's config table](live-trading-runbook.md)). Flipping the
  default would make the ordinary config undeployable, and the remedy
  an operator reaches for — writing `compound = false` by hand —
  restores exactly this decay, only now with the operator believing
  they chose it. A default flip on the *research* side would widen the
  parity gap the matrix above already marks ❌.
- On this build `compound = true` is not a working alternative anyway.
  It compounds into the gross-exposure limitation below and produced
  the −112 % drawdown named in the previous paragraph. A default that
  can drive modelled equity negative is worse than one that quietly
  de-risks.

What changed instead is the silence. `qe_run` now measures, for every
fill that opens a position, the scale factor `compound = true` would
have applied and `compound = false` did not —
`equity_at_that_bar / initial_capital` — and reports the range across
the run. When the spread reaches **1.25×** the finding ships as a
`sizing_basis_decay` entry in `provenance.warnings`: the summary
block's warning banner, the `results.json` artifact, and the HTML
report all carry it. Below that it stays quiet, deliberately — a
warning that appears on every profitable artifact would devalue the
price-level lint it shares the banner with.

Two things the measurement does *not* claim. It is silent on a run
that never opened a position, because "no entries" is not "no drift";
`entries: 0` is reported as unmeasured, never as `1.00×`. And 1.25 is
a judgement, argued at
`include/qe/analytics/sizing_drift.hpp` — the 2026-08-01 study's 2.6×
has to fire, an ordinary +10 % year must not.

### Neither runtime resizes an open position

There is no partial add and no partial trim in either the backtest or
the live engine. A position is entered once and exited once. Every
write path to an existing holding is guarded by "not already in
position".

This is fine until `signalize_universe(..., exit_rank = N)` is used
with `exit_rank > top_k`, which is the whole point of hysteresis. Then:

- `target_gross` scales the names in the **current** top `top_k`, and
  only those.
- A name that has slid to rank `top_k+1 … exit_rank` is still held,
  keeps the notional it entered with, and sits entirely outside the sum
  being scaled.
- A full `target_gross` is then allocated across the fresh top `top_k`
  **on top of it**.

So **`target_gross` bounds the selection, not the book.** The bounds,
derived rather than measured:

- with even weights, gross runs at `target_gross × exit_rank / top_k`;
- at most `target_gross + (exit_rank − top_k) × max_weight`.

For `top_k = 10, exit_rank = 12` that is 1.2× invested under even
weights, and `target_gross + 2 × max_weight` in the worst case. The
engine emits a load-time diagnostic naming both bounds when you
configure this, under both `equal_k` and `inverse_vol` weighting.
Nothing enforces either one.

(An earlier draft of that diagnostic quoted `exit_rank × max_weight` as
the ceiling. It is unreachable: it needs every open leg at the cap at
once, and the current selection cannot be, because the allocator scales
it to exactly `target_gross`. Only the overhang escapes the sum. A
peak-gross figure from a reproduction that is not in the tree was also
removed from three documents rather than published — it had not been
re-measured.)

There is no backstop. `MultiPortfolio` lets cash go negative, and the
equity clamp is deliberately skipped for budget-sized entries. **An
equity curve produced under `exit_rank > top_k` is itself levered**, so
it cannot be the evidence that approves the deployment.

The operating rule until a resize path exists: size `target_gross` and
`max_weight` against `exit_rank`, not against `top_k`.

### A third runtime: walk-forward windows — warmed, purge is opt-in

Walk-forward has its own slicing, and it is not the backtest's.

- ~~**No warmup padding.** A window is a hard slice; indicators restart
  from zero state at its first bar.~~ **Fixed (2026-09-17).**
  Each test slice is now extended backwards by the config's own
  indicator warmup — the maximum `min_history` across every leg's
  entry and exit expressions — and trading is suppressed across that
  padding, so the book still starts flat and windows stay comparable.
  `WalkForwardRun` reports `warmup_bars`, and
  `windows_short_of_warmup` counts any window the series was too short
  to warm. **A non-zero count means the estimate is still flattered**
  and must be read before the Sharpe is.
- ~~**No purge, no embargo.** `test_start = train_end`.~~ **Partly
  fixed.** `walk_forward(purge_bars = N)` now drops N bars between
  train and test, belonging to neither slice. **The default is 0**,
  which reproduces the old behaviour exactly — so an unchanged config
  still has `test_start = train_end`, and a 252-day-lookback factor
  still scores its first test bar from history the optimizer just
  trained on. Purge is available; it is not automatic, and nothing
  will tell you that you did not set it.

The measurements that motivated the fix, kept because they calibrate
how much this mattered: on a 252-day factor with a 2-year test window,
**256 of 503 test bars (50.9 %) were dead** before the first fill, so
the strategy traded only the second year of every window. Moving the
series start by one year, changing nothing else, moved out-of-sample
Sharpe **0.878 → 0.461**; a control with a 20-bar warmup moved
1.110 → 1.075. Sharpe was also mechanically deflated by `√(1−f)` for a
dead fraction `f` — predicted 0.7008 for `f = 0.509`, measured 0.7030 —
and train and test had *different* dead fractions (33.5 % vs 50.9 %),
so the in-sample-versus-out-of-sample delta was not interpretable
either.

**What to carry forward.** A long-lookback factor is now warmed, but it
is still **unpurged unless you say so**. Set `purge_bars` to at least
the factor's lookback before reading a walk-forward number as
out-of-sample, and check `windows_short_of_warmup == 0`.

---

## Data limitations

### There is no point-in-time universe. Survivorship bias is not handled.

This is the largest un-modelled source of error in the whole system,
and it applies to **every** result the engine produces — strategy and
benchmark alike.

A universe in a `.qe` config is a list of tickers you wrote down today.
The engine has no notion of which securities existed, were listed, were
in an index, or were investable at any past date. It cannot have one:
there is no point-in-time constituent data anywhere in the project, and
the only bundled data sources cannot supply it.

Three mechanisms, all of them working against you:

1. **You pick the list knowing the answer.** Anyone assembling a
   sector universe in 2026 is assembling it from names that mattered in
   2026. Backtests on such a list are measuring, in part, your
   knowledge of the outcome.
2. **Failed companies are absent, not weighted at zero.** A ticker that
   was delisted cannot be fetched. It does not appear as a −100 %
   position; it simply is not there.
3. **Late listings silently shorten your study.** Multi-symbol data is
   aligned by **strict intersection** of timestamps. Add one name that
   IPO'd in 2020 to a 2015-start study and the whole study becomes a
   2020-start study, for every symbol. The load only errors when the
   intersection is *empty*; a large truncation is silent. Check the bar
   count in `results.json` against the window you asked for.

How much does it matter? In one 30-name sector study, the equal-weight
buy-and-hold benchmark returned 35.00 % CAGR — and **a single name was
43 % of its terminal value**, with two names accounting for 61 %. That
benchmark is a 2026 list held from 2015. The number is arithmetically
correct and is not evidence about anything.

If you need survivorship-free results, this engine cannot give them to
you today. What it can do is make the bias visible: run leave-one-out
over your largest contributors and report the spread.

### Yahoo data: what the adjustment does and does not do

The bundled fetcher uses the Yahoo Finance v8 chart endpoint. It is a
free, undocumented, unofficial endpoint with no SLA. It rate-limits, it
changes, and it is not a market-data vendor.

- **Prices are on a total-return basis.** Per bar,
  `ratio = adjclose / close`; `close` becomes `adjclose` and
  `open` / `high` / `low` are multiplied by the same ratio, so all four
  columns stay on one basis. This matters because `next_open` fills at
  an open that must be comparable to the close the signal used.
- **The ratio is the dividend component only.** Yahoo pre-applies split
  adjustment before sending, so there was never a split jump to remove.
- **`volume` is left exactly as sent** — a split-adjusted share count,
  not rescaled by `1/ratio`. A dividend does not change share count, so
  rescaling would fabricate a volume trend of up to +43 % at the oldest
  bar of a high-yield name, inside a variable every factor expression
  can read. The trade-off: the market-impact model slightly
  *overstates* friction on a dividend payer early in the window, which
  is the safe direction.
- **A non-payer is bit-identical** to the raw quote.
- **Two fetches of the same symbol over the same window do not agree —
  set `cache_dir` to pin them.** Measured 2026-08-01: the same config
  run twice, nothing else changed, differed on 2 888 of 2 908 equity
  bars, and 21 of 30 symbols came back with a different SHA-256 over
  identical bar counts and identical timestamps. Yahoo recomputes the
  dividend back-adjustment between fetches. The divergence is small —
  around 1e-7 relative — so it will not move a conclusion that rests on
  anything larger, but an unpinned Yahoo result is **reproducible to
  about six significant figures, not bit-exactly.**

  Since 2026-08-06, `cache_dir` on a `yahoo(...)` or
  `yahoo_template(...)` source fixes this: the raw response is stored
  under a filename derived from the request, and a later run with the
  same request replays those bytes. Verified end to end — two runs
  against a warm cache produced results.json files differing only in
  `runtime_ms` and the config's own path and hash, with every bar,
  metric and trade identical.

  **An unset `cache_dir` still does not cache**, deliberately: the same
  provider serves the dashboard's live chart and quote polling, and a
  cache defaulted on would freeze those. Set it on anything whose
  numbers you intend to quote.

  Before that date the argument was accepted, recorded in the
  provenance block, and ignored — so a run could name a `cache_dir` in
  the very block whose purpose is establishing reproducibility, having
  pinned nothing. `csv_url(...)` always cached and was unaffected.
- The load **fails**, naming the symbol, if `adjclose` is missing on a
  settlement interval, disagrees in length with the quote arrays, or
  produces a ratio outside `(0, 1.5]`. It never returns a mix of
  adjusted and raw bars.

Local CSVs default to `format = "yahoo"` and get the same treatment, so
a downloaded CSV and a live fetch of the same symbol agree. Pass
`format = "generic"` for data already on the basis you want.

### Other data facts worth knowing before you are surprised

- **`end` is required in practice.** Nothing substitutes today's date.
  A config with no `end` is rejected rather than quietly extended.
- **The remote-CSV cache never expires.** `csv_url(...)` is fetched
  once and cached under a content-hashed filename; later runs read the
  cached file. Delete it to force a re-fetch. It caches
  **unconditionally and never expires**.

  `yahoo(...)` / `yahoo_template(...)` also cache, but **only when
  `cache_dir` is set** (`src/net/yahoo_provider.cpp:219-222`). That
  changed on 2026-08-06; until then `cache_dir` on a Yahoo source was
  accepted and ignored, and the `YahooCacheDirIgnored` lint that said
  so has been removed. Earlier revisions of this page called `csv_url`
  the only source that caches — it is not, it is the only one that
  caches whether you ask or not.
- **`ibkr_historical(...)` is not implemented** as a backtest source.
  It fails at parse time with a message saying so. The IBKR path is
  live trading only.
- **No corporate-action model beyond the price adjustment above.** No
  spin-offs, no mergers, no ticker changes, no delisting events.
- **Provenance proves reproducibility, not correctness.** Every
  `results.json` carries a SHA-256 per symbol over the *aligned* bars
  the engine actually saw. That lets you prove two runs consumed
  identical data. It says nothing about whether the data was right.
  Worth knowing what it is for: the first thing those digests
  established in practice was that two Yahoo fetches of the same
  window *disagree*. Use them before quoting any result as
  reproducible. A provenance block naming a `cache_dir` does now pin
  the Yahoo body it fetched (since 2026-08-06), but only for runs made
  after that cache was populated — it cannot retroactively pin a
  window fetched before it existed.

---

## Current live warmup limits

Until 0.4.0 the daemon asked its broker for `"60 D", "1 day"` — a
literal, identical for every config ever written, about **41 trading
bars**. A config needing more did not fail. It started, logged nothing
unusual, and evaluated its factor on NaN.

That failure mode is worse than an empty book. NaN sorts **last** in
the cross-sectional pre-pass, so a long-only `top_k` selection quietly
picked from whichever handful of symbols happened to be warm — not
nothing, a *wrong* selection, with no diagnostic anywhere.

It also silently capped research. A deployed strategy's own header
documented living inside the constant rather than fixing it: a 50-day
moving average "cannot run live at all".

### What happens now

1. **Requirement.** Walk every leg's entry, exit and sizing
   expressions — including the inner expression of a cross-sectional
   site, and including an inverse-vol weighting's risk expression — and
   take the maximum warmup. Record which indicator drove it.
2. **Request.** Convert trading bars to a calendar duration —
   `ceil(bars × 365.25 / 252) + 10` days, expressed in whole years past
   ~300 days, because that is what gateways reliably accept. The
   request is sized from the *soft* target (see below), so an `ema`
   config asks for enough history to converge even though it will not
   be refused for missing it.
3. **Verify.** Compare what arrived, per symbol, against the *hard*
   floor. **Refuse the start if any symbol is short.**

Step 3 is the one that matters. Steps 1 and 2 make the right request;
only step 3 makes a wrong answer loud. A holiday cluster, a recent
listing or a pacing violation can all return fewer bars than asked for,
and every one of those used to be a warning followed by a start.

The refusal names the required bars, the driving indicator and the leg
it came from, what was requested, and per short symbol what arrived
with its earliest and latest timestamp. `qe_daemon validate` prints the
requirement without contacting a broker.

A failed fetch for one symbol is a **short delivery**, not a separate
category. It refuses too.

### The limits that remain

- **Daily bars only.** The warmup request is hard-coded to `"1 day"`
  bars. An intraday live strategy is not warmed at its own resolution.
- **IIR settling is advisory, not enforced.** The requirement has a
  hard floor (below which the strategy emits NaN) and a soft target
  (enough bars for an `ema` / `rsi` to hold the state a long-history
  backtest holds). Only the floor refuses. A 200-day EMA wants roughly
  1 400 extra bars and no venue owes anyone six years of dailies, so
  falling short logs a warning that states the residual and starts.
- **`sma` drift is not a warmup problem** and no budget fixes it — see
  the [four classes](#indicator-state-after-warmup-four-classes) above.
- **A recent listing cannot be warmed.** If a name has 90 bars of
  history at the venue, no phrasing of the request produces 252. The
  strategy either shortens its longest lookback or drops the name.
- **The requirement is computed from the config, not from the venue.**
  It does not know that a particular symbol is thinly quoted or that
  the gateway will pace you.

---

## Reconciliation and restart behaviour

### Drift blocks submissions

Reconciliation used to log at WARN, append a notice, and keep trading —
the policy the reconciler's own header documented as "report drift,
don't try to fix it". On 2026-07-31 a daemon booked ten fills, its own
reconcilers contradicted all ten within ninety seconds, and it went on
submitting against a book the account did not hold.

Since 0.4.0, drift **blocks new order submissions** until an operator
acknowledges it. Three things arm the block:

| Source | When |
|---|---|
| Startup reconcile | journal-replayed positions vs the broker's, at start |
| Latched position drift | the running fill reconciler, on its 30-second tick |
| Uncorroborated fill | the order said `Filled`, the execution stream did not account for the shares |

Startup-only arming would not have caught the incident that motivated
this: that daemon started clean and drifted three seconds into the
session.

The gate sits **outside** the order router, so a blocked intent never
reaches the pre-trade risk projection and cannot move the projected
book. The latch starts **held** and is released exactly once, after the
startup evaluation — a future edit that skips that evaluation costs a
daemon that will not trade, rather than one that trades unreconciled.

Operator side:

```bash
qe_daemon reconcile          # the report; exit 3 while blocked
qe_daemon reconcile-ack <block-id> --decision=<d> [--note="..."]
```

`<d>` is one of `broker_is_correct`, `journal_is_correct`,
`resolved_externally`, `accept_risk`. The decision is mandatory and is
written to the journal. The report names, per symbol: the local
quantity, the broker quantity, the delta, when each side was observed,
the **journal offset** of the row the local side came from, and what
you could do about it.

Four properties to know before you rely on this:

- **Exits are blocked too, and there is no honest way around it.** An
  exit is sized against the position whose quantity is exactly what the
  block is about. Gating entries but not exits would be gating on a
  number just declared untrustworthy. A daemon holding positions under
  a block is **not managing them**. Your remedies are `cancel-all`
  (never gated), the venue's own interface, and acknowledging.
- **Nothing is auto-adopted.** An ack records a judgement and re-enables
  submissions. It does not repair, flatten or copy either book, and the
  journal row says `"adopted": false` explicitly.
- **An ack does not outlive the process.** Restarting re-measures the
  condition, and if it is still true a new block is raised. Within one
  process an acknowledged symbol is watermarked so the poll cannot
  immediately re-raise it. A persisted "ignore forever" flag is exactly
  what made the advisory policy useless.
- **An inconclusive reconcile does not block.** A failed
  `list_positions()` is not evidence the books disagree, and blocking on
  it would hand a broker's nightly gateway restart a veto over the next
  session. It is logged at ERROR and the session is described as
  unreconciled.

Cancels, `cancel-all` and the kill switch are unaffected. This is an
additional gate, not a replacement for any of them.

### Restarts keep the day's loss

Both session-equity trackers used to hold their baseline in memory
only, so every restart re-anchored from scratch:

- the **account** tracker at whatever the first equity poll reported —
  and the broker's account summary is an async subscription that
  routinely has not arrived yet, in which case the baseline became
  *current* equity and the day's loss so far read as zero;
- the **strategy** tracker at `execution(capital = ...)`, against a
  fill-driven ledger that also restarted flat at exactly that capital.
  That one is the tracker the daily-loss kill switch actually reads,
  and its restart behaviour was not merely an under-report: it reported
  **zero** loss, deterministically, for the rest of the session.

A daemon that had already lost money came back believing it had not —
wrong in the unsafe direction, on the number that gates a one-way kill.

Since 0.4.0 the daemon journals both baselines and restores them on
replay, keyed to the session they were taken in, so a genuine 16:00 ET
rollover still re-anchors normally.

Two design points that are load-bearing:

- **The strategy ledger and its anchor are restored together or not at
  all.** Restoring the anchor alone is *worse* than restoring nothing:
  a $92,000 anchor against a book that restarted flat at the default
  $100,000 of capital reports an $8,000 **gain** on a strategy that is
  down, and the
  mirror case manufactures a loss large enough to trip a one-way kill
  on a daemon that lost nothing. Cash, shares, anchor and session key
  travel in one journal row and are adopted in one step. A checkpoint
  written under a different `execution(capital = ...)` is refused
  outright.
- **A restored baseline the previous run guessed at stays
  provisional.** It is flagged as such, said out loud at WARN on
  startup, and still upgraded the moment the broker's summary lands. An
  older, closer-to-session-start guess beats re-guessing at the restart
  point, but it is not a brokerage statement and does not claim to be.

Marks are deliberately **not** in the checkpoint. The historical warmup
re-seeds every universe symbol's mark before the restore runs, so a
restored book is valued at today's prices; persisting stale marks would
hide an overnight gap until the first quote landed.

### Restart limits that remain

- **A daemon that dies between a fill and its journal append restores
  the pre-fill ledger.** The checkpoint is written on every fill and
  every session anchor — every moment its fields can change — but the
  window is not zero. The fill reconciler re-anchors the *risk
  projection* against broker truth once a minute and deliberately does
  not touch this ledger, because adopting broker share counts without
  their cost basis would invent equity, and this number gates a one-way
  kill.
- **The strategy book is never seeded from broker positions**, by
  design, for the same reason.
- **The kill switch is one-way within a process** and sticky for the
  process lifetime.
- ~~**Connection-callback teardown on the warmup refusal is narrowed,
  not closed.**~~ **Closed** in v0.4.1 (`2ec0c15`).
  `set_connection_event_callback` now takes a `unique_lock`, assigns
  the slot, then waits on `connection_event_idle_` until
  `connection_event_in_flight_ == 0` — so a detach cannot return while
  a dispatch is still running. It skips the wait only when the caller
  is itself in `connection_event_dispatchers_`, so a handler clearing
  its own slot cannot self-deadlock. The wider second callback is
  detached on the same discipline.
- ~~**`rank` / `quantile` disagree with everything else about an
  infinite member.** Those two still branch on `isnan` while the
  normalizers and both selectors branch on `isfinite`, so an infinity
  gets a rank but no z-score. Narrow, and it does not affect selection
  counts.~~ **Fixed** — both now branch on `isfinite`, and both are
  expressed against the scored block rather than the whole universe.
  The "narrow" reading undersold the second half: `quantile` divided
  the rank by the universe size, so every unusable member cost the
  scored names a bucket at the top they could no longer reach — five
  scored among ten symbols at `q = 5` capped the best name in the
  universe at bucket 2, and `quantile(x, 5) == 4` selected nobody.
  Tests at `tests/test_xs_normalize.cpp:736`. **This changes numbers**
  for any config whose universe carries a NaN or an infinite member on
  some bar; see the CHANGELOG entry before comparing a re-run against
  an old result.

  The `is_top` **under-selection** that stood here until v0.4.0 is
  **fixed**: selection now runs over the finite window
  (`src/dsl/evaluator.cpp:989-999`), so `is_top(xs_zscore(x), k)`
  names the `k` best finite members rather than `k − m`, and selects
  nothing when fewer than `k` are finite. Tests at
  `tests/test_xs_normalize.cpp:626,702`. Note the direction of the
  change: a screen that was quietly under-exposed now takes the full
  `k`, so re-check sizing against the release's upgrade note 6
  before assuming it is a no-op.

---

## Breaking changes

See [Migration and incident history](release-history.md) for this reference.

??? info "Moved section links"

    - <a id="040-qe_daemon-refuses-a-price-level-volatility-factor"></a> [0.4.0 — qe_daemon refuses a price-level volatility factor](release-history.md#040-qe_daemon-refuses-a-price-level-volatility-factor)

    - <a id="040-a-config-whose-warmup-cannot-be-delivered-refuses-to-start"></a> [0.4.0 — a config whose warmup cannot be delivered refuses to start](release-history.md#040-a-config-whose-warmup-cannot-be-delivered-refuses-to-start)

    - <a id="040-reconciliation-drift-blocks-submissions"></a> [0.4.0 — reconciliation drift blocks submissions](release-history.md#040-reconciliation-drift-blocks-submissions)

    - <a id="040-resultsjson-portfolio-writer-is-schema_version-6"></a> [0.4.0 — results.json portfolio writer is schema_version 6](release-history.md#040-resultsjson-portfolio-writer-is-schema_version-6)

    - <a id="040-win_rate-and-avg_win_loss-are-null-not-00"></a> [0.4.0 — win_rate and avg_win_loss are null, not 0.0](release-history.md#040-win_rate-and-avg_win_loss-are-null-not-00)

    - <a id="040-strategymin_history-removed-c-api"></a> [0.4.0 — Strategy::min_history() removed (C++ API)](release-history.md#040-strategymin_history-removed-c-api)

    - <a id="versioned-incident-notes"></a> [Versioned incident notes](release-history.md#versioned-incident-notes)

    - <a id="2026-06-to-2026-07-31-six-weeks-on-the-wrong-factor"></a> [2026-06 to 2026-07-31 — six weeks on the wrong factor](release-history.md#2026-06-to-2026-07-31-six-weeks-on-the-wrong-factor)

    - <a id="2026-07-31-ten-fills-reported-as-booked-against-a-flat-account"></a> [2026-07-31 — ten fills reported as booked against a flat account](release-history.md#2026-07-31-ten-fills-reported-as-booked-against-a-flat-account)

    - <a id="2026-08-04-an-empty-answer-that-was-not-an-empty-book"></a> [2026-08-04 — an empty answer that was not an empty book](release-history.md#2026-08-04-an-empty-answer-that-was-not-an-empty-book)

    - <a id="2026-08-01-a-host-reboot-and-the-watchdog-behaving-correctly"></a> [2026-08-01 — a host reboot, and the watchdog behaving correctly](release-history.md#2026-08-01-a-host-reboot-and-the-watchdog-behaving-correctly)

    - <a id="2026-08-01-a-0-win-rate-published-beside-a-475-return"></a> [2026-08-01 — a 0 % win rate published beside a +475 % return](release-history.md#2026-08-01-a-0-win-rate-published-beside-a-475-return)

    - <a id="2026-08-01-a-drawdown-comparison-that-was-not-like-for-like"></a> [2026-08-01 — a drawdown comparison that was not like-for-like](release-history.md#2026-08-01-a-drawdown-comparison-that-was-not-like-for-like)

## See also

- [Live-trading safety model](live-trading-safety.md) — the design
  rationale for every gate.
- [Live-trading runbook](live-trading-runbook.md) — the operational
  procedures, including upgrading an existing deployment.
- [`.qe` language reference](qe-language.md) — the lint, the escape
  hatch, and every `execution(...)` knob named above.
- [`results.json` schema](results-json.md) — the `provenance` block and
  what `not_computed` means.
- [Walk-forward validation](walk-forward.md) — the harness whose
  cold-start behaviour is described above.
