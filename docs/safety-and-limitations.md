# Safety and known limitations

Everything on this page is something the engine gets wrong, does not
model, or models in a way that will surprise you. It is written down in
one place because the alternative — leaving it distributed across
header comments, commit messages and the memory of whoever found it —
is how the incidents at the bottom of this page happened.

Three rules this page is written under:

- **No number appears here that was not measured.** Where a figure
  comes from reading source rather than running something, it says so.
  An earlier round of work published bounds that had been reasoned
  about rather than measured, and they had to be retracted; the rule
  exists because of that.
- **A limitation stays on this page until it is fixed**, not until it
  is understood or worked around.
- **Anything found in a live deployment gets a dated entry** in
  [Versioned incident notes](#versioned-incident-notes), including the
  part that was embarrassing.

If you are about to trade real money, read
[the safety model](live-trading-safety.md) and
[the runbook](live-trading-runbook.md) as well. This page is the list
of what those two do not protect you from.

---

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
[six weeks on the wrong factor](#2026-06-to-2026-07-31-six-weeks-on-the-wrong-factor).

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
| Warmup history | full series from `from` | derived from the config; **refuses to start if short** | ✅ since 0.3.4 |
| Cross-sectional rank algorithm | `cross_sectional_prepass` | same function | ✅ |
| Cross-sectional batch trigger | every bar, all symbols present | quorum-or-timeout barrier; stale slots on timeout | ⚠️ by design |
| Entry / exit pre-pass | yes | yes | ✅ |
| Sizing pre-pass and warmup | yes | yes | ✅ since 0.3.4 |
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
| **Exact after 0.3.4** | `rolling_vol`, `rolling_zscore`, `realized_vol` | bit-identical | the ring was summed in *physical slot order*, so the result depended on how many bars had been seen. Now summed oldest-first |
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

### A third runtime: walk-forward windows warm nothing

Walk-forward has its own slicing, and it is not the backtest's.

- **No purge, no embargo.** `test_start = train_end`. The first test
  bar is the bar immediately after the last training bar, so a factor
  with a 252-day lookback scores that bar from 252 bars the optimizer
  just trained on. Every boundary leaks a year of history forward.
- **No warmup padding.** A window is a hard slice; indicators restart
  from zero state at its first bar.

Measured consequence on a 252-day factor with a 2-year test window:
**256 of 503 test bars (50.9 %) are dead**, before the first fill. The
strategy trades only the second year of every window. In one study that
meant it sat out a Q4 selloff, a crash, an entire bear market and one
whole year — by calendar accident, not by design, and the resulting
equity curve looks like risk management.

The instability this produces is easy to demonstrate: move the series
start by one year, changing nothing else, and out-of-sample Sharpe went
**0.878 → 0.461** on the same strategy. A control with a 20-bar warmup,
invested in both schedules, moved 1.110 → 1.075.

Sharpe is also mechanically deflated by `√(1−f)` for a dead fraction
`f`, because flat bars contribute exact zeros — predicted 0.7008 for
`f = 0.509`, measured 0.7030. And train and test slices have *different*
dead fractions (33.5 % vs 50.9 %), so the in-sample-versus-out-of-sample
delta that walk-forward exists to produce is not interpretable either.

**Do not use walk-forward to validate a long-lookback factor on this
build.** The number it produces is not an out-of-sample estimate. A
short-lookback factor (tens of bars against a multi-year window) is far
less affected, but still unpurged.

The fix is known and not built: hand the test slice
`[test_start − min_history, test_end)` and score only from
`test_start`. The warmup number already exists — it is the same
function the live daemon now uses.

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
- **Two fetches of the same symbol over the same window do not agree,
  and there is no cache to pin them.** Measured 2026-08-01: the same
  config run twice, nothing else changed, differed on 2 888 of 2 908
  equity bars, and 21 of 30 symbols came back with a different SHA-256
  over identical bar counts and identical timestamps. Yahoo recomputes
  the dividend back-adjustment between fetches. The divergence is small
  — around 1e-7 relative — so it will not move a conclusion that rests
  on anything larger, but it does mean **a Yahoo-sourced result is
  reproducible to about six significant figures, not bit-exactly.**
  `cache_dir` does not help: it is accepted on the `yahoo(...)` source
  and recorded in the provenance block, but no cache is implemented on
  that path, so the run re-fetches. Setting it now raises a config
  lint. If you need a pinned input, fetch once and load the CSV from
  disk with `file(...)`, or use `csv_url(...)`, which does cache.
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
  cached file. Delete it to force a re-fetch. **This is the only source
  that caches** — `cache_dir` on a `yahoo(...)` source is accepted and
  ignored, which is where the belief that Yahoo caches comes from. See
  the Yahoo section above.
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
  reproducible — including one whose provenance block names a
  `cache_dir`, which on a Yahoo source pins nothing.

---

## Current live warmup limits

Until 0.3.4 the daemon asked its broker for `"60 D", "1 day"` — a
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

Since 0.3.4, drift **blocks new order submissions** until an operator
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

Since 0.3.4 the daemon journals both baselines and restores them on
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
- **Connection-callback teardown on the warmup refusal is narrowed, not
  closed.** The refusal path detaches its connection-event callback
  before unwinding, but the connection's read worker copies the
  callback under its mutex and invokes the copy outside it, so a detach
  racing an in-flight dispatch does not stop that dispatch. A second,
  wider callback is also installed later in startup, and the guard is
  armed before that one exists. Narrow window, real, and open.
- **`is_top` under-selects when part of the universe has no score.**
  `xs_zscore` correctly returns NaN for a non-finite member, but
  `is_top`'s rank comparison still counts excluded members in the
  universe size, so `is_top(xs_zscore(x), k)` selects `k − m` names when
  `m` members are non-finite, instead of the `k` best finite ones.
  Under-selection, never over-selection. It needs a decision about what
  `top_k` means when part of the universe has no score, not a patch.
  Relatedly, the ordinal primitives (`rank` / `quantile` / `is_top` /
  `is_bottom`) branch on `isnan` while the normalizers branch on
  `isfinite`, so the two disagree about an infinite member.

---

## Breaking changes

### 0.3.4 — `qe_daemon` refuses a price-level volatility factor

**This is the one that will stop a running deployment from restarting.**

A daemon restarted on a 0.3.4-or-later build **refuses to start** on a
config that applies `rolling_vol` to a bare price variable. Both
`qe_daemon validate` and `qe_daemon start` exit **4**, before any socket
to the venue is opened. `qe_run` is unaffected and still loads such a
config, because reproducing a legacy result has to keep working.

Why it is fatal rather than loud: this warned at load time for a full
release, and a deployment carried the warning in its startup log every
session and ran on it for six weeks.

**Migration — do this before you restart on a new build.** Either:

```qe
// Before — refused at the live boundary
rolling_vol(close, 20)

// Option A: you meant return volatility. A DIFFERENT factor.
realized_vol(close, 20)

// Option B: you meant dollar dispersion. Exact identity, same numbers.
rolling_vol(price_level(close), 20)
```

- **Option B is a no-op.** `price_level(x)` returns `x` and allocates no
  state. The arithmetic is byte-for-byte unchanged, so results,
  rankings and fills are identical. You do not need to re-run your
  gates. Use this if dollar dispersion is genuinely what you want, and
  the config now says so on the page.
- **Option A is a different strategy.** `realized_vol` measures
  something else, will select different names, and needs its gates
  re-run from scratch. Do not treat it as a rename.
- **Option C**, if you need the daemon up right now:
  `--allow-price-level-vol`. It starts and prints the whole refusal at
  ERROR every session. Use it to buy time, not as a resolution.

Verify before you restart:

```bash
qe_daemon validate path/to/live_config.qe   # exit 0 = deployable
```

### 0.3.4 — a config whose warmup cannot be delivered refuses to start

Previously the daemon requested a fixed ~41 bars and started regardless
of what came back. It now derives the requirement from the config and
**refuses when any symbol is short**. See
[Current live warmup limits](#current-live-warmup-limits).

This is breaking in the operational sense: a config that started
yesterday can refuse today. It was not working yesterday — it was
trading on NaN-sorted rankings — but the failure is now visible where
it used to be silent. A recent listing in your universe is the most
likely trigger.

### 0.3.4 — reconciliation drift blocks submissions

Drift that previously produced a WARN and continued trading now stops
all new submissions, entries **and exits**, until acknowledged. See
[Reconciliation and restart behaviour](#reconciliation-and-restart-behaviour).
A supervisor script that assumed a daemon reporting healthy is a daemon
that will trade needs to check `qe_daemon reconcile` (exit 3 while
blocked) or the `reconcile_blocked` field on `status`.

### 0.3.4 — `results.json` portfolio writer is `schema_version` 6

Was 4; 5 was already taken by walk-forward, since schema numbers are one
namespace across all four writers rather than a per-writer counter. The
only difference is the added `provenance` block. Every v4 field is
emitted unchanged with the same meaning, so a reader that accepted v4
accepts v6. The dashboard reads both, and degrades a malformed
`provenance` block to "absent" rather than failing the load.

### 0.3.4 — `win_rate` and `avg_win_loss` are `null`, not `0.0`

On portfolio, multi-symbol and multi-strategy runs these two metrics are
not computed — round-trip trades do not pair cleanly across legs — and
they used to serialize as `0.0` and print as `Win rate 0%`. They are now
`null` in JSON and `—` on the console, and `provenance.not_computed`
names the metric and the reason.

If you have a consumer that reads these fields, it must handle `null`.
Note also that a multi-symbol **walk-forward** still reports `0.0` here:
that value feeds the parameter-selection argmax, so changing it changes
which cell an optimizing run picks. That is an optimizer change, not a
reporting fix, and it is deliberately not in this release. Provenance
says so on that path rather than claiming a null the writer does not
produce.

### 0.3.4 — `Strategy::min_history()` removed (C++ API)

A pure virtual on the strategy-runner interface plus CRTP hooks on both
strategy bases. Nothing in the engine, daemon, reports or live path ever
called it — warmup has always been enforced by NaN propagation — and a
warmup number nobody reads is worse than none, because it reads as a
guarantee. The DSL-level `qe::dsl::min_history()` stays and is now what
sizes the live warmup. Only affects code embedding the C++ library.

---

## Versioned incident notes

Dated by the period the defect was live, oldest first — which is not
the order they were found in; the first two were both uncovered on
2026-08-01, weeks after they started. Each says what was wrong, how it
was found, and what changed.

### 2026-06 to 2026-07-31 — six weeks on the wrong factor

**What happened.** A paper deployment ran a strategy whose selection
factor was `rolling_vol(close, 20) * -1.0`, described in its own config
and every downstream note as a low-volatility factor. It is not one; see
[Semantic traps](#semantic-traps). It is approximately a low
share-price tilt. Every selection the deployment made in those six
weeks was made on the wrong measurement.

**How it was found.** Not by an alert. By a factor-audit pass that
measured what the function returns and compared it to what the name
claims.

**Why nothing stopped it.** The load-time lint fired on every single
startup, into a log nobody was reading.

**What is important about it.** The recorded gate results for that
strategy are real measurements *of that factor*. They were not
retracted. What was wrong was the name, and every sentence downstream
that reasoned from the name.

**What changed.** The lint became a hard refusal at the live boundary
(exit 4), and the finding now travels with the artifact — into
`provenance.warnings`, onto stdout in the run summary, and into a banner
above the KPI cards in the HTML report. A warning that only exists in a
log is not a control.

**A related fact, kept separate on purpose.** That deployment's own
journal records no confirmed fills at all in those six weeks. So "traded
the wrong factor for six weeks" is accurate about what the strategy
selected.

> **Correction, 2026-08-04.** This paragraph continued "and wrong about
> what the account ever held", inferring an empty account from the
> journal's silence. That inference is withdrawn. The journal fact
> stands — it records no confirmed fills — but the broker's book
> contains holdings that appear to predate the 07-31 session, and a
> journal that never saw a fill is not evidence that none occurred.
> Their origin is an open question, not a settled one.

### 2026-07-31 — ten fills reported as booked against a flat account

> **Correction, 2026-08-04. The headline claim on this section was
> wrong, and the error is more instructive than the incident.** The
> account was not flat. Every piece of evidence that said it was came
> from two broker queries that are now known to return a confident
> empty answer regardless of the truth — see "an empty answer that
> was not an empty book" below. The daemon's ten fills were real.
> The section is kept, retitled, because the reasoning failure it
> now documents is worth more than the incident it originally did.

**What was believed.** Between 09:30:31 and 09:30:33 the daemon wrote
ten `order_fill` events and ten `position` events — its first ever. Its
own reconcilers appeared to contradict them within ninety seconds: nine
projection drifts at 09:31:04 ("gate held 4, broker holds 0 with 0 still
in flight"), ten position drifts at 09:32:04 (`booked=N broker=0
drift=-N`). A direct account query the next day returned no positions
and no working orders. That was read as confirmation.

**What was actually true.** Every `broker=0` in those reports is the
same defect: `decode_position_data` was reading a field POSITION_DATA
does not contain, so every row failed to parse, and the frame handler
dropped failures silently. `list_positions()` returned success with an
empty vector. The reconcilers were not contradicting the fills — they
were comparing the journal against nothing and rendering the nothing as
zero. The follow-up "no working orders" is worthless for a second,
unrelated reason: that query is client-scoped and was issued from a
different client id than the one that placed the orders.

The fills were real. The broker's book holds seven of those symbols at
exactly the journal's quantities, and for each one the broker's average
cost exceeds the journal's fill price by exactly the $1.00 per-order
commission — arithmetic that cannot arise from positions that were never
opened.

**What is still unexplained.** One symbol recorded **both** an
`order_fill` and an `order_rejected` in the same second — broker code
2161, the price-cap notice, whose text ends "you will not receive a
fill". One order cannot be both, and the decoder defect does not account
for it.

**The lesson worth keeping.** Three independent controls agreed the
account was flat, and all three were reading the same broken sensor. The
agreement felt like corroboration and was not: a defect upstream of
every consumer produces unanimity, not contradiction. Nothing in the
system distinguished "the venue says zero" from "we failed to parse what
the venue said", because the code path that discarded the failure was a
bare `return`.

**Root cause.** The connection synthesised fills from order-status
cumulative-fill transitions and discarded the execution-details stream
entirely, on the reasoning that the two carry the same facts. They do
not: order status is the order's state, execution details is the broker
saying shares changed hands. Discarding one leaves nothing to disagree
with on the day they differ, and the disagreement is the whole signal.

**What changed.** Executions are decoded, logged with their execution
id, exchange and account, and counted per order (0.3.3). A shortfall
between what the order claims and what the executions account for is now
**durable state**, not a log line: it is counted, exposed on `status`,
flips the reconcile verdict off "clean", and raises a submission block
(0.3.4).

That work still stands on its own reasoning — order status is the
order's state, execution details is the broker saying shares changed
hands, and discarding one leaves nothing to disagree with on the day
they differ. But it should be read as a deliberate tightening, not as a
remedy for an observed loss, because the loss did not occur. Note also
that the tiebreaker named in the original version of this entry — "the
broker's own position, which the reconciler queries" — was for the whole
of that period a query that always answered zero.

### 2026-08-04 — an empty answer that was not an empty book

**What happened.** `decode_position_data` read a `primaryExchange` field
that IBKR's POSITION_DATA frame does not carry. Every subsequent field
shifted by one, so `position` and `avgCost` were parsed from
non-numeric text, every row failed, and the frame handler discarded each
failure with a bare `return` — no log line, no counter. `POSITION_END`
then sealed the empty map and `list_positions()` returned **success**
with an empty vector. Every consumer was told the account held nothing:
startup position seeding, the reconcile gate, the risk projection, the
dashboard's position panels.

The account held twelve positions, including a short in a long-only
strategy that the empty book had been concealing.

**Why it survived.** Both tests covering the decode encoded the same
phantom field. They agreed with the decoder and passed. A test written
from the same misreading as the code confirms the misreading.

**What changed.** The field is gone, the tests are rewritten against a
live wire capture, and a row that fails to decode no longer shrinks the
book — it marks the whole snapshot UNKNOWN, the same as a timeout, and
logs the raw frame. Two consumers were corrected alongside it: startup
seeding no longer converts a short into a long by taking `abs(qty)`, and
the dashboard's reconciler no longer treats an absent expectation as an
expectation of zero.

**What it should change in how you read this page.** Any claim on this
page about what the account held, dated before 2026-08-04, was reading
this sensor. Two entries above have been corrected in place; both
corrections are marked. Treat the remainder with the same suspicion —
a defect this far upstream does not announce which conclusions it
touched.

Two defects of the same shape are still open in the same file — a decode
failure discarded with a bare `return`, and a snapshot that seals a
partial result as a complete one — on the open-orders and account-value
paths. The open-orders one is the more dangerous of the two: it makes
"we could not determine what is open" indistinguishable from "nothing is
open", to a caller whose entire job is cancelling whatever is open.

### 2026-08-01 — a host reboot, and the watchdog behaving correctly

Recorded because a control that works is also worth dating. A host
reboot took the daemon down mid-session; the watchdog escalated as
designed — soft pause at 15:51:24, kill on broker disconnect at
15:51:57.

### 2026-08-01 — a 0 % win rate published beside a +475 % return

**What happened.** Every portfolio run reported `"win_rate": 0.0` and
`"avg_win_loss": 0.0` in `results.json` and printed `Win rate 0%` on the
console — on runs with 938 to 1318 trades and returns above +400 %. A
research batch published tables carrying it.

**Root cause.** Not a miscalculation. The metric is deliberately not
computed on that path, because pairing round-trip trades across legs
does not decompose cleanly. That reasoning is fine. Emitting `0.0` for
it is not: the JSON schema already carried `null` elsewhere and the
console already rendered non-finite values as `—`, so "not computed" was
expressible, and this path chose the one value indistinguishable from a
real result.

**Why it is on this page.** A 0 % win rate beside a +475 % return is
absurd enough to catch by eye. The same pattern on a metric with a
plausible zero would not be, and there is no reason to think this was
the only instance of the pattern.

**What changed.** `null` in JSON, `—` on the console, and
`provenance.not_computed` names the metric and the reason. One path
(multi-symbol walk-forward) still reports `0.0`, deliberately, because
that value feeds parameter selection — provenance says exactly that
rather than claiming a null.

### 2026-08-01 — a drawdown comparison that was not like-for-like

**What happened.** A validation matrix reported a strategy's maximum
drawdown as −14.29 % against a buy-and-hold benchmark's −48.13 %, and
read that as the strategy's main advantage. A walk-forward pass then
found that `execution(compound = ...)` defaults to false and no config
in the study set it, so the strategy's risk per dollar of equity decayed
2.6× across the sample while the benchmark's did not.

**What it means.** The drawdown comparison was between a book running at
roughly 40 % of the benchmark's risk at the moment being compared. Under
walk-forward, where every segment restarts un-decayed, the same
strategy's pooled drawdown was −38.82 % against the benchmark's
−36.90 % — no advantage at all.

**What was and was not retracted.** No published number was restated;
they are all arithmetically correct measurements of what was run. The
*interpretation* was withdrawn. The return comparison survives — the
strategy did not beat buy-and-hold on return either way — and the
drawdown comparison does not.

**What changed.** The `compound` interaction is documented in the
[parity matrix](#compound-true-has-no-live-equivalent) above. A
like-for-like fully-invested comparison is still not expressible on this
build, because `compound = true` interacts with the gross-exposure
limitation and produced a −112 % drawdown in the same study. That is an
open gap, not a fixed one.

---

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
