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
| `execution(whole_shares = true)` | applied — floors, then the same four-decimal quantum | applied | ✅ since 0.5.5 |
| `execution(compound = true)` | scales entries by `equity / initial` | **no equivalent** | ❌ |
| Resizing an open position | never | never | ✅ (and both are [wrong](#neither-runtime-resizes-an-open-position)) |
| Fills | modelled at the next open/close, plus fixed commission and slippage in bps | whatever the venue does | by definition ❌ |
| Fill price vs the modelled open | exact | **+22.07 bps adverse**, notional-weighted | ❌ [measured](#what-execution-actually-cost) |
| Commission | `commission_bps` + `commission_per_order_usd` per fill | IBKR `clamp($0.005/share, $1.00, 1% of notional)` | ⚠️ [measured](#what-execution-actually-cost) |
| One order, many executions | one fill | **41.3% of orders**, up to 7 | ❌ not modelled |
| A venue price cap that does not fill | fills unconditionally | IBKR may cap a market order and not fill | ❌ [adversely selected](#what-execution-actually-cost) |
| An order that never reaches the venue | cannot happen | `submit_order timed out waiting for ack` | ❌ not modelled |

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

### `whole_shares`: modelled on both paths since 0.5.5, and what it costs

`execution(whole_shares = true)` floors each order to an integer share
count. Some venues and accounts reject fractional orders outright, so it
is not optional there.

**Both paths model it as of 0.5.5, and that is new.** Until then the
backtest ignored the flag entirely — it appeared zero times in
`include/qe/backtest/`, `include/qe/strategy/` and `apps/cli/` while the
live engine floored every entry, so a backtest that justified a
deployment measured a book the daemon would not buy. EPIC-107 moved the
rule into `qe::execution::rounded_qty`
(`include/qe/execution/share_rounding.hpp`) and put both paths through
it.

Two consequences that are easy to get backwards:

- **The flag now selects a grid, not whether rounding happens.** Left at
  its `false` default, an entry is still quantised — to four decimals,
  the granularity the live path has always used. Before 0.5.5 the
  backtest carried full `double` precision there, so the two disagreed
  on every fractional entry of every config, including configs that
  never mention `whole_shares`.
- **A run recorded before 0.5.5 is not comparable to one after it.** The
  numbers moved by design. Nothing in the engine can tell you which side
  of that line an old `results.json` came from except its
  `provenance` block.

**The size of that gap is measured, and which figure you want depends
on the question.** Measured 2026-10-29 on
`deploy_it_lowvol_rb1_25k_paper_pipeline.qe` — the workspace file, which
sets `rebalance = 1` — over 1,275 entries (EPIC-98; that file names the
raw inputs and scripts):

| | |
|---|---|
| mean capital left idle, per entry | **5.12%** of the $833.33 per-leg budget ($42.68 an entry; $54,419.28 over the window) |
| what that idle capital actually cost | **0.59%** of profit ($524.63 of $89,466.01) |
| in total return / CAGR | 357.8640% → 355.7655% (−2.10 pp) / −0.0454 pp |
| entries that could not afford one share | 0 |

**That is the rb=1 file, not the book that is live, and what separates
them is turnover.** The host has run `rebalance = 20` since 2026-09-21:
EPIC-98 H-1 found the daemon loading a release snapshot
(`releases/.../incumbent.qe`) rather than the workspace file, with
`strategy_id` pinned to the rb=1 basename by PER-265, so `status` prints
an rb=1 name over an rb=20 config. The config its `ExecStart` named on
2026-11-02, `deploy_it_lowvol_rb20_25k_paper_nominal.qe`, additionally
sets `price_basis = "nominal"`, which the measured file does not. The
per-entry idle fraction is set by share price against a leg's budget
(the year-by-year relation below) and plausibly carries over; the
**totals do not**. The $54,419.28 idle, the
$524.63, the 0.59% and the −2.10 pp of total return are all proportional
to the 1,275 entries rb=1 produced, and rb=20 does not produce 1,275
entries. **It has now been re-measured, on both.**

**Re-measured in-tree, 2026-11-03** —
`research/results/epic107_wholeshare_drag_2026_11_03.md`, with 3,199
per-entry rows in the `.csv` beside it. Six backtests, three
pairs, each pair differing only in `whole_shares`. This supersedes the
2026-10-29 figures above, whose inputs lived outside the repository:

| | rb=1, engine basis | rb=1, nominal | rb=20, nominal |
|---|---|---|---|
| drag, % of profit | **0.5960%** | **3.7081%** | **3.4299%** |
| drag, dollars | $518.03 | $797.30 | $901.46 |
| mean idle, % of leg budget | 5.1253% | 6.3528% | 5.9543% |
| entries | 1,274 | 1,468 | 457 |
| entries that cannot afford one share | 0 | 0 | 0 |

The rb=1 engine-basis column reproduces the 2026-10-29 measurement to
0.0035 pp on idle and 0.0096 pp on drag through a different code path,
which is why the other two columns are worth believing.

**The percentage is six times larger on the nominal basis while the
dollars barely move** — $518 against $797. Flooring did not get more
expensive; the profit it is a fraction of shrank, from $86,923 to
$21,501, because the nominal basis removes the split look-ahead that the
engine basis feeds the ranking. A drag quoted as "% of profit" therefore
says as much about which basis produced the denominator as about
flooring.

**Flooring does not cost risk-adjusted performance.** Max drawdown falls
in all three pairs; Sharpe rises on both nominal pairs and moves by
−0.0003 on the engine basis, which is noise at that precision. Less
capital deployed is less capital exposed. So `whole_shares = false` buys
return, not safety.

**Every figure in that table is a lower bound, and the reason is the
subject of this whole page's neighbour.** The engine floors each order
against Yahoo `adjclose` — split-adjusted by the vendor, then
dividend-back-adjusted by the engine. **A live order floors against the
nominal price.** Undoing the dividend half alone moves the drag from
0.5864% to 0.6521% of profit (measured; the dividend ratio over the
1,275 entries runs 0.6772 / 0.9358 / 1.0000 for min / mean / max). The
split half is **still unmeasured, but it is not unmeasurable, and this
page said otherwise until 2026-11-03.** Its *sign* is known, because a
split multiplier is ≥ 1 by construction, so the real drag is higher than
the figures above by an amount nobody here has measured yet.

**And `price_basis = "nominal"` does not close it, which is worth stating
because the naming invites the opposite conclusion.** EPIC-106 made the
split table per-bar and EPIC-107 ran the deployed config on the nominal
basis, so it would be reasonable to assume the split half is now
measured. It is not. `price_basis` moves what the *factor reads*, not
what an order *floors against*: measured on 2026-11-03, the largest
fractional entry quantity is `1813.0883` on the nominal basis and
`1813.0884` on the engine basis — the same number to seven significant
figures, against a flooring price of about $0.46, which is NVDA deep in
split adjustment and not a price that ever traded. Every figure in the
table above floors against the adjusted close.

The retracted claim was "no local cache carries events for these 30
names", resting on a record of 64 `events=div,split` requests answered
with HTTP 429 and zero successes. The 429 record is real but belongs to
a research script's own separate fetching attempt; it is not the path the
engine uses, and it does not say what it was read as saying.
**The engine's chart request already carries `&events=split` in the same
response as the bars** — `src/net/yahoo_provider.cpp` states it as "Same
response, no extra request", and it is how `qe_run` measures a symbol's
split factor at all. Audited 2026-11-03 across the eight Yahoo cache
directories on this machine: 2,923 cached bodies, 2,491 of them with no
`events` block at all (written before the request asked for them, which
is how the belief formed), and **87 symbols with split events present —
including every splitter in the deployed universe**: NVDA and ANET and
APH with two each, AAPL, AVGO, FTNT, IBM, KLAC, LRCX and NOW with one.

Two caveats keep this from being a finished measurement. The caches are
**gitignored** (`data/yahoo_cache/` has zero tracked files), so nothing
here reproduces from the tree alone — a re-fetch is required, and
`SplitBasis::requested` is what distinguishes "measured, no splits" from
"nobody asked". And the drag itself still has to be recomputed with the
rounding done against the corrected basis, which is what EPIC-106's
per-bar factor is for.

An earlier version of this section said "about 14% of capital left
idle". That figure was a **single-bar snapshot** from the deployed
config's own header rather than a measurement over the run — but the
rewrite that replaced it went too far in saying it "does not
reproduce", and went too far in the reassuring direction. Recomputed on
the last bar of that window (2026-07-28) it is 12.49% across all 30
names and 8.61% across the ten actually held; over the window, on the
basis the engine uses, it is 5.12%. **Floor every entry at the
right-edge price level instead — which is nearer to what a live order
faces than `adjclose` is — and mean idle is 13.167% with a drag of
7.915% of profit.** That is where a 14%-shaped number comes from. It is
a different question, not a wrong answer.

What the old number is *not* is a split correction: reaching a 14% drag
by scaling the flooring price uniformly needs a factor of 19.77 — a
~20:1 split on all 30 names — at which point 828 of the 1,275 entries
cannot buy a single share. The sensitivity table is in the artifact
below.

So there are three numbers and they answer three questions: **5.12%**
idle / **0.59%** of profit on the engine's own basis, a lower bound;
**13.167%** / **7.915%** on the right-edge level, the closest thing
here to a live floor; and the old **~14%**, a one-bar snapshot that
happens to land near the second. Quote the first only with the words
"on the backtest's basis" attached.

The 5.12% and the 0.59% differ ninefold for a reason worth knowing.
The score is `rolling_vol(price_level(close), 20) * -1.0`, which ranks
roughly by inverse share price, so it **selects the cheap names —
exactly where integer rounding costs least**. One NVDA round trip
entered at a flooring price of $0.46 is 58.2% of all profit and has an
idle fraction of 0.005%; excluding it, the drag is 1.40% of the
remaining profit. Weighting idle by gross PnL rather than by entry
gives 2.20%, not 5.12%.

**Do not read the 0.59% as a constant — it tracks share price.** Mean
idle per entry went 1.22% (2015) to 11.43% (2025) to 12.13% (2026) as
the mean entry price went $19.86 to $225.08, and the 2024-and-later
entry cohort pays 3.23% of its own profit. The relation that fits all
12 year-rows is `mean idle ≈ mean_price / (2 × budget)`: the drag is set
by how large one share is against a leg's budget, and nothing in the
backtest knows that ratio exists.

**Past 100% the failure mode is silence.** On the last bar of that
window MU closed at $820.53 against an $833.33 budget. One further 1.6%
rise and that leg floors to **zero** shares and simply never enters —
no error, no warning, and the backtest still holds it.

So a live book ran below its backtest, and no gate saw it, because the
gate is the backtest. **Since 0.5.5 the backtest floors too, so the gate
now sees it** — a current run of a `whole_shares = true` config reports
the floored book, and the advice that used to follow ("treat every
backtest return as an overstatement") no longer applies to it.

What does still apply: the zero-share case above is silent in both
runtimes. A leg whose budget cannot afford one share submits nothing and
the position simply never opens, with no error and no warning, on the
live path and in the backtest alike. The backtest agreeing with the
daemon is not the backtest telling you. Size positions so each leg
comfortably clears one share, and read the per-leg budget lines the
startup gate prints as the ceilings they are.

Provenance: every figure above reconciles with
`research/wholeshare_drag_2026_10_29_results.txt`, which is the
measurement's own output, copied into the repo at EPIC-104 time. It had
been quoted from a job scratch directory outside the tree — the same
unreproducible footing this section criticises the ~14% for standing
on.

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

### What execution actually cost

Until 2026-07-31 this project had never observed a fill, so every cost
figure in every gate was a model. The deployed `ibkr-paper` book has
since filled 89 times across 10 sessions, and its own journal prices
that model. Measured 2026-10-29 over 46 orders / 89 executions /
$35,118.66 of notional (EPIC-98 X-1..X-5):

| | modelled | realized | |
|---|---|---|---|
| Slippage | 3 bps | **22.07 bps** notional-weighted | **7.4x** |
| Commission | 10 bps | **13.32 bps** | 1.33x |
| Combined | 13 bps/side | **35.4 bps/side** | 2.72x |

Re-running `deploy_it_lowvol_rb1_25k_paper_pipeline.qe` at the realized
cost takes rb=1 Sharpe from 1.2090 to 1.1363 and costs 5.5% of profit —
larger than the `whole_shares` drag above, which has its own section.
The live book rebalances every 20 bars rather than every bar, and EPIC-98
ran that pair as well: Sharpe **1.2404 → 1.2193**, a loss of 0.0211 where
rb=1 loses 0.0727. No profit-share figure was published beside the rb=20
pair, so the 5.5% has no counterpart there.

**The slippage is adverse, not drift.** Buys fill *above* the official
open (mean +30.50 bps) and sells fill *below* it (mean −16.80 bps). A
market that merely moved would push both the same way; this pushes them
apart. 49 of 89 executions were worse than the modelled 3 bps.

**The commission gap is a per-order minimum, not a rate.** IBKR charges
`clamp($0.005/share, $1.00, 1% of notional)`, and at this book's per-leg
budget the $1 minimum always binds — structurally, not as a sample: the
per-share component beats $1 only below `0.005 × 833.33 = $4.17` a
share, and the cheapest name in the universe trades at $71.93. 13 of 26
orders came in within a cent of exactly $1.00. `commission_bps` is
proportional and cannot express this at any value;
`commission_per_order_usd` can and has since PER-257, but a config that
leaves it at 0 charges nothing for it. On
`deploy_it_lowvol_rb1_25k_paper_pipeline.qe` (`rebalance = 1`) that is
2540 fills × $1 = **$2,540 on a $25,000 book, 10.2% of capital**, and
modelling it moves max drawdown from 19.5459% to **19.9797%** against a
gate threshold of 20%. Both of those are fill-count figures on an rb=1
run. The $1 minimum still binds per order at rb=20 — that part is
structural — but 2540 fills is not an rb=20 number, and the 0.02 pp of
margin against the 20% gate has not been re-measured at the frequency
the book actually trades.

**Partial fills are real and common.** 19 of 46 orders (41.3%) filled in
more than one execution, up to seven — one `SELL` of NOW on 2026-09-16
went out as seven one-share executions. Not journal duplication: for
every order carrying fee rows, `n_fill_rows == n_commission_rows ==
n_distinct_exec_ids`. The backtest produces one fill per slice and has no
concept of this.

**A venue price cap can turn a market order into no fill at all.** The
live router submits `OrderType::Market` unconditionally
(`src/live/ibkr_order_router.cpp:85` — "v1 — Market only"), and IBKR caps
such an order into a limit order under its own regulatory obligations;
one 2026-07-31 `BUY 2 KEYS` came back as code 2161 saying so and did not
fill. The backtest's `fill_model = "next_open"` fills unconditionally.
**That non-fill is adversely selected**: a cap binds precisely when the
price gapped away, so the entries it drops are the ones that moved, and
the backtest counts them all as filled at the open. One occurrence in 10
sessions, so the magnitude is unmeasured — the mechanism is not.

Separately, one `order_submit_failed` on 2026-08-05 reports
`submit_order timed out waiting for ack`, on a **SELL** — an exit that
never went out. The backtest has no path where a submission fails.

All of the above is recomputed by `tools/qe-realized-cost.py`, which
reads the journal and a reference-price CSV and emits a JSON receipt.
Its numbers are not a snapshot in this document; run it. The tool also
reports what it could not price, rather than treating a missing
reference as zero slippage.

#### Three cost sets, and which question each answers

The numbers above are easy to mix up with the ones in a config or in a
gate policy, because all three are bps of notional. They are not
interchangeable, and only one of them is a measurement of anything.

**1. Modelled — an assumption you chose.** `execution(commission_bps,
commission_per_order_usd, slippage_bps)` in a `.qe` file. It decides
what the backtest charges and nothing else; the live path never reads
these three fields (it reads `capital`, `whole_shares`, `compound` and
the two schedules). Use it to ask "what would this strategy have done
at that friction". Two places hold modelled sets on purpose:

  - The deployment gate's `cost_floor` / `stress_cost_floor`
    (`research/deploy/usd5000-*-live-gate.json`) are **policy minima a
    receipt has to meet**, not costs anything paid. The stress floor's
    `slippage_bps >= 22` is a static backstop derived from set 2 below.
  - A **frozen research screen's** cost cells stay where they were
    frozen. A screen whose numbers move is a different screen, so these
    do not get updated when a measurement arrives.

As of EPIC-104 T104.6 the deployment preparation configs in
`research/deploy/*.qe` are the exception: their base cells now carry set
2's measured values rather than 10 bps / 3 bps, because they are the
starting point for new research and not a frozen result.

**2. Paper-measured — real fills, from a simulator.** 22.07 bps
slippage and 13.32 bps commission, 46 orders / 89 executions /
$35,118.66 of notional over ten sessions, from one `ibkr-paper` book's
own journal. This is the only execution cost this project has ever
measured, and it is what prices set 1. `tools/qe-realized-cost.py`
recomputes it.

**It does not predict live slippage, and should not be read as a
conservative estimate of it either.** The fills came from IBKR's paper
simulator, not from a venue: no queue position, no real counterparty,
and the paper runtime requests delayed data, so these fills "do not
establish real-time executable spreads or real-money slippage"
([strategy comparison](strategy-comparison.md)). It is also measured on
ten mid/large-cap US equities — transplanting it to ETFs is
conservative by construction, as
`research/friction_22bp_2026_11_02_protocol.md` argues, but
transplanting it to another venue, account type or order size is
unvalidated **in either direction**.

**3. Live-measured — does not exist.** No real-money account has placed
an order from this project. There is no live slippage number, no live
commission number, and nothing here is an estimate of one. Until a
funded trial produces them, the stress scenario is a static backstop
rather than a bound on observed cost, and the gate's forward check —
"whatever the trial actually paid, the stress scenario must be at least
that pessimistic" — is the mechanism that will replace it.

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

There is now a backstop, and by default it is off. Until 2026-11-02
nothing in the tree called `validate_intent` — the gross / net /
per-symbol check that `include/qe/backtest/constraints.hpp` describes in
its first paragraph. That header and its own unit tests were the only
places the check existed, which reads like protection and is worse than
no check at all (EPIC-104 `D-5`). As of T104.5 `MultiEngine` calls it in
the per-bar loop, before an intent becomes a next-bar fill, and counts
every refusal in `MultiEngine::constraint_report()`.

Three things to know before relying on it:

- **It is opt-in, and the engine holds every check at zero until a
  caller opts in** (`PortfolioConstraints::disabled()`). That is
  deliberate: a refused intent does not fill, so arming a check moves
  equity, trade count and every metric derived from them, and every
  study recorded before 2026-11-02 has to keep reproducing. A zero in
  `constraint_report().refused` is therefore also what an opted-out run
  reports — ask `constraints_enabled()`, not the counter.
- **A `.qe` config cannot turn it on.** `MultiEngine::set_constraints(...)`
  is a C++ API; no `execution(...)` or `backtest(...)` kwarg reaches it
  and `qe_run` never calls it. Today only an embedder or a test can arm
  it, so **every number `qe_run` produces is still unchecked for
  leverage.**
- **The documented defaults — gross ≤ 2.0, net ≤ 1.0, per-symbol ≤ 25% —
  are what `PortfolioConstraints{}` gives you**, not what you get by
  leaving it alone.

Separately from the check, `MultiPortfolio` lets cash go negative and the
equity clamp is deliberately skipped for budget-sized entries. **An
equity curve produced under `exit_rank > top_k` is itself levered**, so
it cannot be the evidence that approves the deployment — and that holds
for any run that did not arm the constraint.

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

### The point-in-time universe exists. The data cannot feed it, so survivorship bias is still not handled.

This is the largest un-modelled source of error in the whole system,
and it applies to **every** result the engine produces — strategy and
benchmark alike. The heading used to read "there is no point-in-time
universe"; EPIC-99 built one, and the limitation moved from the engine
to the data rather than going away.

By default a universe in a `.qe` config is a list of tickers you wrote
down today. The engine has no notion of which securities existed, were
listed, were in an index, or were investable at any past date **unless
you ask for one**. `universe(as_of = ...)` takes a universe's members from a
point-in-time snapshot instead of a list typed today; see
`docs/qe-language.md`.

That changed in EPIC-99 and the honest summary is that the mechanism
works and **the data cannot support it** — read the measurement below
before deciding this problem is solved. Until EPIC-99 this section said
"there is no point-in-time constituent data anywhere in the project".
That was wrong even then: `data/universe/sp500_pit_membership.json` has
held 19 annual S&P 500 snapshots (30 June, 2008 through 2026) and 425
departed names for some time, and nothing pointed at it.

Three mechanisms, all of them working against you:

1. **You pick the list knowing the answer.** Anyone assembling a
   sector universe in 2026 is assembling it from names that mattered in
   2026. Backtests on such a list are measuring, in part, your
   knowledge of the outcome.
2. **Failed companies are absent, not weighted at zero.** A ticker that
   was delisted cannot be fetched. It does not appear as a −100 %
   position; it simply is not there.
3. **Late listings shorten your study — loudly now, silently before.**
   Multi-symbol data is aligned by **strict intersection** of
   timestamps. Add one name that IPO'd in 2020 to a 2015-start study and
   the whole study becomes a 2020-start study, for every symbol.

   Since EPIC-99 T99.6 any loss is warned about and a loss of 50 % or
   more is an error naming the member that bound it. Before that the
   load only errored on an *empty* intersection and a large truncation
   was silent, so this used to end with "check the bar count in
   `results.json` against the window you asked for" — advice that put
   the burden on the reader for a check the loader can do.

   The threshold is measured, not chosen. Across four real research
   universes (93 symbol-loads) the loss is **0.0000 exactly** — same-venue
   US equities share one calendar — while adding RIVN, ARM or SNDK to
   AAPL+MSFT over 2015..2026 costs 58 %, 74 % and 86 %. There is no
   tolerance setting: a study that genuinely wants the short common
   window should narrow its own date range, which says so where a reader
   of the config will see it.

How much does it matter? In one 30-name sector study, the equal-weight
buy-and-hold benchmark returned 35.00 % CAGR — and **a single name was
43 % of its terminal value**, with two names accounting for 61 %. That
benchmark is a 2026 list held from 2015. The number is arithmetically
correct and is not evidence about anything.

#### What a point-in-time universe actually costs you

`universe(as_of = ...)` removes mechanism 1 above by construction: the
members come from the snapshot, not from your memory of 2026. It does
nothing about mechanism 2, and that is where it fails.

Measured 2026-10-29 against the 2014-06-30 snapshot (501 names,
2015 Q1 window, the engine's own endpoint —
`tools/qe-pit-universe-reachability.py` reproduces it):

| | names | share |
|---|---|---|
| in the snapshot | 501 | 100 % |
| of those, departed from the index since | 202 | 40.3 % |
| still fetchable from Yahoo today | 347 | 69.3 % |
| **not fetchable** (126 x HTTP 404, 28 x HTTP 400) | **154** | **30.7 %** |
| of those 154, on the file's `departed` list | 147 | 95.5 % |

**The loss is not spread across the universe. It is the departed
population** — precisely the names whose absence is the bias. A run that
skipped them would hold 347 names with 55 departed members (15.9 %) and
present as point-in-time against 40.3 % in the snapshot: about a quarter
of the gap closed, reported as all of it. That is why an unfetchable
member fails the run and why there is deliberately no tolerance setting.

Two further findings make 69.3 % an upper bound rather than a floor:

- **The repo's `data/yahoo_cache/departed/` does not rescue it.** 263 of
  its 425 bodies are cached HTTP 404s; 158 carry bars. Best case — live
  Yahoo plus every usable byte of that cache — is 368/501 (73.5 %) and
  only 76 of the 202 departed names (**37.6 %**).
- **Tickers are not stable identifiers.** Of the 21 names only the cache
  could supply, the cached windows begin in 2018, 2021, even 2026: the
  ticker has been reassigned to a different company (ACT, APC, FB).
  Substituting those prices into a 2015 universe is worse than missing
  them, because nothing downstream could detect it. A point-in-time
  universe wants CUSIP/ISIN/PERMNO, and a price API keyed on ticker
  cannot provide them.

So: **if you need survivorship-free results, this engine still cannot
give them to you** — not because the universe is missing, but because
62 % of the names that would make it survivorship-free are unobtainable
from the bundled source. What remains is a data-source decision, not an
implementation one. Meanwhile, what the engine can do is make the bias
visible: run leave-one-out over your largest contributors and report the
spread.

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

  **That inherited split adjustment is look-ahead for any factor that
  ranks on a price LEVEL, and it is large.** Yahoo divides every
  historical bar by the splits that came *after* it, so bar `i`'s level
  is a function of corporate actions in `i+1..T`. For returns it is
  correct and harmless, which is why the engine does not undo it. For a
  level it is not: NVDA's oldest bar over 2015..2026 sits at 0.0239 of
  the price that traded, a factor of 40 across two splits. On
  `deploy_it_lowvol_rb1_25k_paper_pipeline.qe` (`rebalance = 1`), moving
  to the as-traded basis took Sharpe from 1.2082 to 0.9975 and put it
  *below* a never-rebalanced equal-weight basket of the same 30 names
  (1.1067) — the comparison its own
  [deployment gate](deployment-gate.md) cites. **That ranking flip is an
  rb=1 result and does not reproduce at the frequency the live book
  trades.** EPIC-99's own appendix re-ran the same factor, universe,
  window and costs at `rebalance = 20` and got 1.2408 → 1.1103 — still
  *above* the basket, by 0.0036, which is indistinguishable from zero.
  The drop happens at both frequencies; the reversal is rb=1 only, and
  EPIC-99 says in as many words that it should not be hung on the
  deployed config. `universe(price_basis = "nominal")` moves the
  level back toward the price that traded; `qe_daemon` refuses to start a
  `price_level` factor left on the vendor's basis. See
  `docs/qe-language.md`.

  **The correction is per bar since EPIC-106, and that is what makes it
  right.** `f_s(i)` is the product of the splits whose ex-date is after
  bar `i`, so each bar gets the factor that bar needs: ×40 at NVDA's
  oldest 2015 bar, ×10 after the 2021 split, and **×1 at the newest
  bar**, where the vendor's close already *is* the price that traded.

  - **At the right edge the two bases now agree, and that sentence was
    false a day before it was true.** Until EPIC-106 the correction was
    a single scalar — the *oldest* bar's — multiplied into every bar, so
    `"nominal"` applied ×40 to a newest bar whose true factor is 1.0 and
    `"engine"` was the as-traded basis there. Worse, the scalar differed
    per symbol, so the right edge was **reordered rather than merely
    rescaled**: each name's score multiplied by its own constant `k`,
    turning a cross-sectional ranking into a ranking by `k·sd20`.
    Measured at the 2026-11-02 close over the deployed 30-name universe,
    the two forms' top-10 differed by **3 names** — the engine basis
    taking AAPL / APH / NVDA where the scalar `"nominal"` took
    ADI / NXPI / TXN (`EPICS/EPIC-104-price-basis-reaches-live.md`, a
    Python reference computation rather than an engine run).

    The history is kept rather than replaced because the two sentences
    look alike and mean different things. EPIC-105 **removed** "at the
    right edge the two bases agree" as false, and as the sentence that
    made seeding a live session with the field look harmless. EPIC-106
    made the right edge genuinely ×1, so the claim holds now — for a
    reason that did not exist then.

  - **What this does NOT retroactively fix: any artifact written before
    2026-11-03.** Those ran the scalar form and carry its numbers.
    `results.json` now reports `price_level_per_bar` beside
    `price_level_factor`, and the series digest folds the per-bar table,
    so a scalar run and a per-bar run over identical bars are no longer
    the same fingerprint — but an older file has neither field and is
    the scalar form by definition.

  - **And the change is bounded by `f_s(oldest)` equalling the old
    scalar.** The per-bar form therefore moves only decisions taken on a
    bar that is not the window's first: a strategy entering once at the
    start sees no difference, while long runs and every rebalance after
    the first bar do.

  - **So the look-ahead is a backtest property, and seeding the live
    engine with this field is not the harmless unblock it looks like.** A
    long historical study on `"engine"` carries the look-ahead above; a
    live session choosing names from a 20-bar window does not, because
    its newest bar's true factor is 1.0 and `"engine"` already gives it
    that. Handing the live engine the scalar as it stands would *add* the
    3-of-10 reordering above, not remove one. `price_basis` is therefore
    honoured by `qe_run` and `qe_factor` and **refused** by the live
    engine (EPIC-104 T104.1) — refused rather than ignored, which is what
    it did until 2026-11-02, but still not usable. Since EPIC-105 the
    refusal is narrower and earlier: narrower because a `"nominal"`
    declaration that no bound expression reads is inert and starts with a
    WARN instead (T105.10), and earlier because `qe_daemon validate` and
    `qe_daemon start` run the same predicate before the broker connect,
    where until 2026-11-03 `validate` answered "config is deployable"
    about a config the engine then refused at startup step (11), after
    the gateway handshake (T105.3). Making the kwarg usable live needs a
    **per-bar** factor so the right edge gets ×1; that is not built.

  Until 2026-11-03 the first bullet said the right edge was where the two
  bases *coincided*, which inverted its own argument: coincidence reads
  as "seeding the live engine is harmless", when the reordering seeding
  would inject had already been measured one EPIC earlier. The conclusion
  — do not seed — never changed. The published reason was backwards.

  One thing this leaves open, and it is the live question: the backtest
  evidence behind a deployed `price_level` strategy was computed over a
  window where most bars *do* have splits ahead of them, so how much of
  its Sharpe came from that is a separate measurement from anything on
  this page. Do not read a forward paper trial as settling it — the trial
  ranks at the right edge, where `"engine"` is already the as-traded
  basis, so it carries none of the look-ahead the historical study does
  and prices none of it.
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
  provider serves the live chart and quote polling, and a
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
- [Forward trial power](forward-trial-power.md) — why a 60-session paper
  trial detects operational failure but cannot resolve a Sharpe: one
  window in five of a strategy with a ten-year Sharpe of +1.27 reports a
  negative one, and the sign of its advantage is a coin flip.
