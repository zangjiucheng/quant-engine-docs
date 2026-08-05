# Factor research

A research path separate from `qe_run`'s event-driven backtest.
The goal: rank symbols by an authored expression every bar, measure
how well that ranking predicts forward returns, and score the
resulting long-short portfolio.

The CLI is `qe_factor`, the input is a `.qe` file that evaluates to
`research(...)`, and the output is `factor_report.json`.

New to the project? [`docs/first-factor-tutorial.md`](first-factor-tutorial.md)
is a 10-minute WKSP → Cmd+S → F7 round trip on SPDR sector ETFs;
this page is the desk reference.

## Quick start

```qe
# example_factor_research.qe
research(
  universe = universe(
    symbols = ["XLK", "XLF", "XLE", "XLV", "XLY",
               "XLP", "XLI", "XLB", "XLU"],
    data    = yahoo_template("1d", "2015-01-01", "2025-01-01"),
  ),
  factors = [
    factor("momentum_20", expr = (close / sma(close, 20)) - 1.0),
    factor("rsi_14_inv",  expr = rsi(close, 14) * -1.0),
  ],
  horizons  = [1, 5, 20],
  quantiles = 5,
  rebalance = 5,
  output    = output(report = "out/factor_report.json"),
)
```

Run:

```bash
./build/release/bin/qe_factor example_factor_research.qe
```

Open `out/factor_report.json`. Each `factors[i]` entry has:

- `ic[]` — one row per `horizons[h]`:
  - `ts_ic_mean` / `ts_ic_std` — per-symbol Pearson IC of the factor vs
    forward return, averaged across symbols.
  - `xs_ic_mean` / `xs_ic_std` — per-bar Spearman rank IC across the
    cross-section, averaged across bars.
  - `xs_ic_t_stat` — `xs_ic_mean / (xs_ic_std / sqrt(n_obs))`. **Do
    not use this for inference.** It is the iid t-stat and it is
    inflated by roughly `sqrt(horizon)`; it is retained only so
    pre-v5 reports stay readable. See
    [Three t-stats, and which one to trust](#three-t-stats-and-which-one-to-trust).
  - `xs_ic_t_stat_nw` — the Newey-West overlap-corrected t-stat.
    **This is the one to act on.** `|t| > 3` is the rough sanity
    threshold for a single-factor study — applied to *this* number.
  - `xs_ic_t_stat_nonoverlap` — the conservative cross-check.
  - `xs_ic_se_nw`, `xs_ic_nw_lag`, `xs_ic_nonoverlap_phases` — the
    machinery behind the two corrected numbers.
  - `n_obs` — bars that contributed (NaN-dropped bars excluded).
- `long_short` — equity-curve backtest of the quintile spread
  (top vs bottom), equal-weighted in each leg, dollar-neutral:
  - `equity[]` / `period_returns[]` / `period_open_ts[]` for plotting
  - `total_return`, `sharpe`, `max_drawdown` (renamed from `max_dd` in
    schema_version 4; the dashboard loader keeps reading the legacy key
    for older reports on disk), `turnover`
  - `holding_periods_per_year` — the annualization cadence `sharpe`
    was computed against, `bars_per_year / rebalance`. **Reports before
    schema_version 5 annualized every Sharpe by a hardcoded 252
    regardless of rebalance cadence**, which multiplied it by
    `sqrt(rebalance)`. See
    [The Sharpe annualization fix](#the-sharpe-annualization-fix).

## Why two ICs

- **Spearman cross-sectional IC** is the right number when comparing
  *different factors* on the same universe — rank correlation strips
  out scale differences (RSI is 0–100, momentum is a ratio, z-scores
  are unbounded).
- **Pearson time-series IC** is the right number when asking whether
  *one factor* has linear predictive power over time inside one symbol.
  This is what classic IC literature reports.

The XS t-stat is the one to act on first — specifically
`xs_ic_t_stat_nw`, not `xs_ic_t_stat`; see the next section. The TS
mean is the secondary check (if XS IC is +0.05 but TS IC is -0.05, the
factor is reranking the cross-section without being a winner inside any
single name — worth knowing).

## Three t-stats, and which one to trust

Short version: **quote `xs_ic_t_stat_nw`. Never quote
`xs_ic_t_stat`.**

The IC at horizon `h` is measured by sampling an `h`-bar forward return
at *every* bar. Consecutive samples therefore share `h - 1` of their
`h` per-bar returns — bar `t` and bar `t+1` are measuring almost the
same thing. The IC series that comes out is heavily autocorrelated, so
dividing by `sqrt(n_obs)` as if the observations were independent
understates the standard error by roughly `sqrt(h)`, and overstates the
t-stat by the same factor. At `h = 60` that is a 2.6× to 7.7×
inflation.

This is not a hypothetical. A 2026-07-29 audit of a factor-research
corpus re-measured 57 recorded conclusions; **every large t-stat in it
was at `h = 60`, and none survived correction.** One sector-ETF panel's
headline `rsi_14_inv` at `h = 60` read `t = +4.80` — a number well past
any "this is real" threshold. Corrected, it is **+1.21** Newey-West and
**+0.64** on non-overlapping subsamples. Across that panel's 48 cells
the largest corrected `|t|` is **1.91**: nothing significant anywhere.

| Field | What it is | Use it? |
|---|---|---|
| `xs_ic_t_stat` | `xs_ic_mean / (xs_ic_std / sqrt(n_obs))`. Treats overlapping windows as independent. | **No.** Retained for backward readability of pre-v5 reports only. |
| `xs_ic_t_stat_nw` | `xs_ic_mean / xs_ic_se_nw`, where the standard error is Newey-West with a Bartlett kernel at truncation lag `min(h - 1, n_obs - 1)`. Models the autocovariance instead of ignoring it. | **Yes — this is the headline number.** |
| `xs_ic_t_stat_nonoverlap` | Splits the IC series into `h` interleaved phases (`p, p+h, p+2h, …`), takes each phase's ordinary t-stat, averages them. Each phase's observations come from genuinely non-overlapping windows. | **As a cross-check.** More conservative; it throws data away rather than modelling the dependence. If it and the NW t disagree sharply, believe the smaller one. |

Two supporting fields make the correction auditable rather than
magic: `xs_ic_nw_lag` is the Bartlett truncation lag actually applied
(`h - 1`, clamped to `n_obs - 1`), and `xs_ic_nonoverlap_phases` is how
many phases contributed (a phase with fewer than 2 observations or zero
variance is skipped; `0` means the estimate is NaN).

**At `h = 1` all three are equal**, exactly and bit-for-bit — there is
no overlap at horizon 1, so there is nothing to correct. That identity
is asserted end-to-end through the JSON, so it is a usable smoke test
on any report you are handed.

Do **not** assume `|t_nw| <= |t_iid|` as a rule. It usually holds, but a
*negatively* autocorrelated IC series shrinks the HAC standard error
below the iid one and the corrected t comes out larger. The correction
is a correction, not a haircut.

The same two corrected statistics are emitted per walk-forward window
(`walk_forward.windows[].xs_ic_t_stat_nw` /
`.xs_ic_t_stat_nonoverlap`). Count "significant windows" on those.

## The Sharpe annualization fix

`long_short.sharpe` is `(mean / sd) * sqrt(annualization)` over
`period_returns`. One element of `period_returns` spans `rebalance`
bars — it is a *holding period*, not a bar. So the annualization factor
must count holding periods per year, `bars_per_year / rebalance`.

Through schema_version 4, qe_factor passed a hardcoded **252**
regardless of cadence. At `rebalance = 20` on daily bars the true
figure is `252 / 20 = 12.6`, so every reported Sharpe was multiplied by
`sqrt(252 / 12.6) = sqrt(20) = 4.47`.

**If you are reading a report with `schema_version` below 5, divide its
`long_short.sharpe` by `sqrt(rebalance)`** — the report's own
`run_meta.rebalance_bars`, not a constant. A `rebalance = 5` report is
off by `sqrt(5) = 2.24`, not by `sqrt(20)`. Getting that divisor from
the wrong config is itself a mistake that happened.

From v5 on, two fields record the convention so nothing has to be
reconstructed: `run_meta.bars_per_year` (from the declared `resolution`
for a `yahoo_template(...)` universe, from timestamp-gap inference for
a `file(...)` one) and `long_short.holding_periods_per_year` (the value
actually passed). qe_factor logs the arithmetic and the resulting √
multiplier on every run, and warns when the cadence leaves fewer than
two holding periods per year.

The change is a numeric **no-op at `rebalance = 1`**.

## Authoring rules

Inside `factor(name, expr = ...)`:

- `expr` is a signal-layer expression, same grammar as
  `signal(entry = ..., exit = ...)`. Per-bar variables (`close`,
  `open`, `high`, `low`, `volume`, `bar_index`) and indicators
  (`sma`, `ema`, `rsi`, `lag_return`, `rolling_zscore`,
  `realized_vol`, …) are all available; see `docs/qe-language.md` for
  the full grammar.
- **`rolling_vol(close, n)` is the std-dev of the dollar price level,
  not of returns.** A factor built on it ranks cheap names, not calm
  ones. Use `realized_vol(close, n)`. qe_factor emits a `config lint`
  line when it sees the trap; do not ignore it.
- The expression must produce a *scalar* per bar — not a boolean.
  Boolean comparisons (`>`, `<`, `cross_above`, …) compile but reduce
  the factor to 0/1, throwing away rank information. Don't.
- `let` bindings at the top of the file work inside factor expressions
  exactly as they work inside signals — useful for shared window
  sizes across multiple factors.

Inside `universe(symbols, data)`:

- `data` must be either `yahoo_template(resolution, start, end)`
  (symbol filled per universe entry) or `file(path)` where `path`
  contains `%s` as the symbol placeholder.
- Symbols are loaded sequentially and aligned to the intersection of
  their trading days. Symbols with very different coverage get
  truncated — Yahoo's cache makes the first run the only slow one.
- **Prices are on a total-return basis.** Yahoo bars (and Yahoo-format
  CSVs) are dividend back-adjusted at load — `close` becomes
  `adjclose`, and `open`/`high`/`low` take the same per-bar ratio.
  `volume` is deliberately left alone; see
  [`docs/qe-language.md`](qe-language.md)'s `yahoo(...)` section for
  why. qe_factor logs one line per run naming how many symbols moved
  and by how much. **Every IC, Sharpe and P&L in this document's
  workflow differs from a pre-EPIC-86 run on dividend-paying
  symbols** — a non-payer is bit-identical.

Inside `research(...)`:

- `horizons` (default `[1, 5, 20]`) — forward-return windows in bars,
  used for IC only.
- `quantiles` (default `5`) — number of buckets for the long-short.
  Must satisfy `2 * quantiles ≤ |symbols|`.
- `rebalance` (default `5`) — holding period for the long-short, in
  bars. The forward return is compounded over this window.
- `output.report` — path for `factor_report.json`. If omitted,
  qe_factor prints the JSON to stdout.

## Workflow

1. Write a `.qe` with one factor and a small universe (3–5 symbols).
2. Run `qe_factor`, eyeball the IC table. If `|xs_ic_t_stat_nw| < 2`,
   the factor isn't doing what you thought — debug the expression in
   isolation first.
3. Expand the universe. Re-run. Keep an eye on `n_obs` —
   cross-sectional alignment drops bars on holiday mismatches.
4. Add 2–3 candidate factors to the same file. Compare their
   `xs_ic_mean` head-to-head.
5. When you find one with `|xs_ic_t_stat_nw| > 3`, a
   `xs_ic_t_stat_nonoverlap` that agrees with it, and a plausible
   `long_short.sharpe`, write it up as a `signal(...)` for the regular
   backtest engine to validate end-to-end (including costs).
   Thresholding on `xs_ic_t_stat` instead is how 57 conclusions were
   recorded and then retracted.

## F7 FCTR dashboard panel

The dashboard has a dedicated screen — **F7 FCTR** — that hot-loads
any `factor_report.json`. Set the path once in **Settings (Cmd+,) →
Research → "Factor report path"** and forget it; the panel mtime-
watches the file, so every `qe_factor` re-run auto-refreshes the
view without restarting the dashboard.

What the panel shows:

- **Top strip** — config path, schema version, report mtime,
  universe size, horizons list, quantiles, rebalance bars. Adds a
  `· stale (<reason>)` badge in amber when the latest reload
  attempt failed (file deleted, malformed JSON, etc.) — the prior
  good snapshot keeps rendering underneath.
- **Factor selector** — dropdown of every factor in the report.
  Selection persists across frames within the session.
- **IC table** — one row per horizon. Columns: horizon, XS IC mean,
  **t (iid)**, **t (NW)**, TS IC mean, n_obs, verdict. Both t-stat
  columns and the verdict are colored, but **the verdict is driven by
  the Newey-West t** (falling back to the iid one only when a pre-v5
  report leaves it unrecorded):
    - **green** when `|t| > 3` ("signal")
    - **amber** when `2 < |t| ≤ 3` ("weak")
    - **dim**  otherwise ("noise")

  Reading the two columns side by side is the point: a wide gap
  between them is the overlap inflation made visible, and on a
  pre-v5 report the NW column renders `n/a` rather than inventing a
  number.

  The verdict text carries a `+` / `−` (U+2212) sign suffix
  (`signal+` / `signal−` / `weak+` / etc.) so you can spot at a
  glance whether the factor is predictive in its natural direction
  or when inverted — a `momentum_20` factor with `t = −3.35` reads
  as `signal−`, telling you the trade is to short the
  high-momentum names, not buy them.
- **Long-short metrics strip** — periods, total return, Sharpe,
  max DD, turnover. Return + Sharpe colored green/red by sign.
- **Equity curve** — ImPlot time series of LS period equity,
  shaded green above the 1.0 baseline (gains) and red below
  (losses). Pan / wheel zoom on the x-axis; the y-axis auto-refits
  to whatever's currently visible.
- **Per-bar XS IC plot** — ImPlot time series of the per-bar
  cross-sectional IC for the currently selected horizon. A horizon
  tab strip above the plot lets you flip between horizons; defaults
  open on whichever horizon has the largest `|t-stat|`. Includes a
  zero reference line and a horizontal mean-IC line. Series with
  `n_obs > 5000` are stride-decimated for rendering (the panel
  surfaces the stride above the plot). Hover the plot to see the
  date + per-bar IC at the nearest sample.

If `factor_report_json_path` is unset, F7 shows an empty-state
pointing back to Settings. If the path is set but the file doesn't
exist yet, F7 shows a "run qe_factor" hint; the panel switches over
the moment the file appears.

The panel reads schemas **v1 through v5**. Lower versions still
load — the panels that need newer fields render an "upgrade by
re-running qe_factor" hint instead of breaking, and every v5-only
field reads as `n/a` rather than `0`. That distinction matters: `0.0`
would render as the verdict `noise`, i.e. as a finding nobody made.
A `schema_version` above 5 is rejected, which surfaces as the amber
`stale` badge over the last good snapshot.

## Walk-forward IC

Full-sample IC averages across the entire history can hide:

- a factor that worked 2015-2020 and broke 2021+ (**regime break**),
- a factor whose IC trends linearly toward zero (**signal decay**),
- a factor whose IC oscillates wildly between +0.2 and -0.2 every
  year (**unstable, not tradeable**).

To check for any of these, opt into walk-forward IC by adding
`walk_forward = walk_forward_ic(window_bars, step_bars)` to
`research(...)`:

```qe
research(
  universe = universe(
    symbols = ["XLK", "XLF", "XLE", ...],
    data    = yahoo_template("1d", "2015-01-01", "2025-01-01"),
  ),
  factors      = [factor("rsi_14_inv", expr = rsi(close, 14) * -1.0)],
  horizons     = [5, 20, 60],
  walk_forward = walk_forward_ic(window_bars = 252, step_bars = 21),
  output       = output(report = "out/wf_report.json"),
)
```

On daily bars, `window_bars=252` is a one-year window and `step_bars=21`
is monthly stride. qe_factor then runs `ic_analysis(...)` per window
per (factor, horizon) and emits a `walk_forward` block inside each
`ic[]` entry of the `factor_report.json` (v3 and up):

```json
"walk_forward": {
  "window_bars": 252,
  "step_bars": 21,
  "windows": [
    {"open_bar": 0, "close_bar": 251,
     "open_ts_ns": ..., "close_ts_ns": ...,
     "n_obs": 232,
     "xs_ic_mean": 0.034, "xs_ic_std": 0.18,
     "xs_ic_t_stat": 2.92,
     "xs_ic_t_stat_nw": 0.71,
     "xs_ic_t_stat_nonoverlap": 0.55,
     "ts_ic_mean": 0.08, "ts_ic_std": 0.05},
    ...
  ]
}
```

The two corrected per-window t-stats arrive with schema_version 5 and
are NaN on older reports (and on any window with `n_obs == 0`). Prefer
`xs_ic_t_stat_nw` here: a window is short by construction, which makes
the non-overlapping estimator very noisy at window scale.

### Reading the F7 panel

The dashboard's F7 FCTR walk-forward panel (below the per-bar XS IC
plot, shares the horizon tab) renders a time series of the per-window
`xs_ic_mean`, plus reference lines:

- **dim horizontal at 0** — no-signal baseline
- **muted horizontal at the full-sample `xs_ic_mean`** — anchor for
  "is this window above or below the headline IC?"
- **rolling line** — colored green / amber / dim by the FULL-SAMPLE
  t-stat bucket (same scheme as the IC table verdict)
- **per-window dots** overlaid on the rolling line — each dot
  colored by THAT window's own t-stat bucket. Lets you spot the
  windows where the signal was strong (green dots cluster) vs
  windows where it broke (dim/amber clusters), even when the
  full-sample line color suggests an even read.
- Hover the plot to see the window's date range, IC mean, t-stat,
  and observation count at the nearest dot.

The header strip shows a trend-slope badge from
`qe::analytics::rolling_ic_trend_slope` (simple OLS of `xs_ic_mean`
vs window index). Green when slope > 0 and R² > 0.3, red when
slope < 0 and R² > 0.3, dim otherwise — keeps the "factor is decaying"
warning from firing on a noisy line with no real trend.

### Reading the JSON yourself

A factor is "real and persistent" when most windows clear `|t| > 2`
**in the same direction**, measured on the corrected t-stat:

```python
import json, math
r = json.load(open("out/wf_report.json"))
for fac in r["factors"]:
    for ic in fac["ic"]:
        wf = ic["walk_forward"]
        ws = wf["windows"]
        # v5+. On an older report this key is absent / null and there is
        # no corrected number to count — re-run qe_factor.
        t = [w.get("xs_ic_t_stat_nw") for w in ws]
        t = [x for x in t if x is not None and not math.isnan(x)]
        pos = sum(1 for x in t if x > 2)
        neg = sum(1 for x in t if x < -2)
        print(f"{fac['name']:14s} h={ic['horizon']:3d}: "
              f"{pos} positive + {neg} negative / {len(ws)} windows")
```

Two mistakes are baked into the version of this snippet that shipped
before schema_version 5, and both of them produced published claims
that had to be retracted:

- **It counted `abs(...)`.** A window at `t = +2.5` and a window at
  `t = -2.5` are not two pieces of evidence for the same factor; they
  are evidence that the factor flipped sign. A "60/108 significant
  windows" headline from that snippet turned out, on audit, to be
  **41 positive + 23 negative** — a factor whose direction reversed in
  a fifth of the sample, not a factor that survives across regimes.
  Print the two counts separately, as above, and read the smaller one
  as noise you are being paid to notice.
- **It counted on `xs_ic_t_stat`**, the iid statistic, which is
  inflated by roughly `sqrt(horizon)`. At `h = 20` or `h = 60` that
  alone manufactures most of a "significant window" count.

So: a factor with, say, 60 positive and 4 negative windows out of 108
on `xs_ic_t_stat_nw` is genuine persistence, and much stronger evidence
than a single full-sample number. The same tally on `xs_ic_t_stat`,
sign-blind, is not evidence of anything.

### Limits

- **Fixed rectangular window only.** Expanding / exponential-decay
  windows are planned.
- **No alarm system.** The slope badge is a visual; there's no
  cron / Slack hook telling you when a factor's IC slope tips
  below a threshold.
- **No per-symbol rolling TS IC.** The aggregate per-window TS IC
  is in the JSON, but the dashboard doesn't yet plot per-symbol
  lines.

## What this layer is NOT

- **Not** a substitute for `signal(...)` + `qe_run` — those are still
  the path for "I want to trade this idea." Factor research is the
  upstream filter that decides what to trade.
- **No sector / beta neutralization.** If you want sector-neutral
  ranking, subtract a per-bar sector mean inside the factor
  expression. Proper Barra-style neutralization is its own can of
  worms.
- **No long-only mode.** The long-short backtest is symmetric. A
  long-only flag is on the deferred list.

## See also

- `docs/qe-language.md` — full `.qe` grammar reference.
- `docs/forecasting.md` — `predict_return(...)` and related ML
  predictors. Factor research and the forecasting layer are orthogonal
  — a research-vetted factor often becomes a feature inside a
  `predict_return(...)` model.
- `apps/dashboard/workspace_examples.cpp` —
  `example_factor_research.qe` is the starter file dropped into fresh
  workspaces.
