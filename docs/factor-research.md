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
  - `xs_ic_t_stat` — `xs_ic_mean / (xs_ic_std / sqrt(n_obs))`. The
    standard "is this factor real?" number — `|t| > 3` is the rough
    sanity threshold for a single-factor study.
  - `n_obs` — bars that contributed (NaN-dropped bars excluded).
- `long_short` — equity-curve backtest of the quintile spread
  (top vs bottom), equal-weighted in each leg, dollar-neutral:
  - `equity[]` / `period_returns[]` / `period_open_ts[]` for plotting
  - `total_return`, `sharpe` (annualized via 252), `max_drawdown`
    (renamed from `max_dd` in schema_version 4; the dashboard loader
    keeps reading the legacy key for older reports on disk),
    `turnover`

## Why two ICs

- **Spearman cross-sectional IC** is the right number when comparing
  *different factors* on the same universe — rank correlation strips
  out scale differences (RSI is 0–100, momentum is a ratio, z-scores
  are unbounded).
- **Pearson time-series IC** is the right number when asking whether
  *one factor* has linear predictive power over time inside one symbol.
  This is what classic IC literature reports.

The XS t-stat is the one to act on first; the TS mean is the secondary
check (if XS IC is +0.05 but TS IC is -0.05, the factor is reranking
the cross-section without being a winner inside any single name —
worth knowing).

## Authoring rules

Inside `factor(name, expr = ...)`:

- `expr` is a signal-layer expression, same grammar as
  `signal(entry = ..., exit = ...)`. Per-bar variables (`close`,
  `open`, `high`, `low`, `volume`, `bar_index`) and indicators
  (`sma`, `ema`, `rsi`, `lag_return`, `rolling_zscore`, …) are all
  available; see `docs/qe-language.md` for the full grammar.
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
2. Run `qe_factor`, eyeball the IC table. If `|t-stat| < 2`, the
   factor isn't doing what you thought — debug the expression in
   isolation first.
3. Expand the universe. Re-run. Keep an eye on `n_obs` —
   cross-sectional alignment drops bars on holiday mismatches.
4. Add 2–3 candidate factors to the same file. Compare their
   `xs_ic_mean` head-to-head.
5. When you find one with `|xs_ic_t_stat| > 3` and a plausible
   `long_short.sharpe`, write it up as a `signal(...)` for the regular
   backtest engine to validate end-to-end (including costs).

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
  t-stat, TS IC mean, n_obs, verdict. The t-stat + verdict columns
  are colored:
    - **green** when `|t-stat| > 3` ("signal")
    - **amber** when `2 < |t-stat| ≤ 3` ("weak")
    - **dim**  otherwise ("noise")

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

The panel reads schemas **v1 / v2 / v3**. Lower versions still
load — the panels that need newer fields render an "upgrade by
re-running qe_factor" hint instead of breaking.

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
`ic[]` entry of the v3 `factor_report.json`:

```json
"walk_forward": {
  "window_bars": 252,
  "step_bars": 21,
  "windows": [
    {"open_bar": 0, "close_bar": 251,
     "open_ts_ns": ..., "close_ts_ns": ...,
     "n_obs": 232,
     "xs_ic_mean": 0.034, "xs_ic_std": 0.18, "xs_ic_t_stat": 2.92,
     "ts_ic_mean": 0.08, "ts_ic_std": 0.05},
    ...
  ]
}
```

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

A factor is "real and persistent" when most windows clear the
`|t-stat| > 2` bar:

```python
import json
r = json.load(open("out/wf_report.json"))
for fac in r["factors"]:
    for ic in fac["ic"]:
        wf = ic["walk_forward"]
        n = len(wf["windows"])
        sig = sum(1 for w in wf["windows"] if abs(w["xs_ic_t_stat"]) > 2)
        print(f"{fac['name']:14s} h={ic['horizon']:3d}: {sig}/{n} significant windows")
```

A factor with 60/108 significant windows on real Yahoo sector data
is a genuine signal that survives across most regimes — much stronger
evidence than a single full-sample `t-stat = +4`.

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
