# Walk-forward validation

`walk_forward(base, train_window=..., test_window=...,
step_window=..., optimize=sweep(...)?)` is the cheapest OOS
sanity-check you can run on a backtest: split the data into
rolling train / test windows, optionally pick the best parameters
on each train slice, and **only report the test-slice metrics**.

This document covers what walk-forward actually proves, how to
read the F4 BCKT view, and the v1 limits to be aware of.

## TL;DR

```qe
let fast = 10
let slow = 50
let base = backtest(
  data     = yahoo("SPY", "1d", "2018-01-01", "2024-12-31"),
  strategy = signal(
    entry  = cross_above(sma(close, fast), sma(close, slow)),
    exit   = cross_below(sma(close, fast), sma(close, slow)),
    symbol = "SPY",
  ),
  execution = execution(capital = 100_000),
  output    = output(results = "out/wf_spy.json"),
)

walk_forward(base,
  train_window = "365d",      # 1 year IS
  test_window  = "90d",       # 3 months OOS
  step_window  = "90d",       # walk by 3 months → non-overlapping
  optimize     = sweep(base,
    axes = [axis("fast", values = [5, 10, 15, 20]),
            axis("slow", values = [50, 100, 150, 200])],
    metric = "sharpe",
  ),
)
```

Result on F4 BCKT:

- **Main equity panel** shows the OOS-only stitched curve with
  vertical grey rules at every test-window boundary, badge
  `· OOS · N windows · optimized|fixed`.
- **Right panel** is `WALK-FORWARD · IS vs OOS`: per-window
  table of IS Sharpe / OOS Sharpe / Δ / OOS Return / chosen
  params. Sweep-optimized runs also draw `PARAM DRIFT` mini-
  charts below the table — one line per axis showing the
  selected value across windows.

## Fixed-parameter rolling validation

`optimize` is optional. Omit it and walk-forward becomes a much
simpler tool: run the *same* fixed parameters on every test slice
and report stitched OOS metrics. No optimizer, no `PARAM DRIFT`
chart, no IS-vs-OOS Sharpe gap (because there's no IS pick).

```qe
let base = backtest(
  data     = yahoo("SPY", "1d", "2018-01-01", "2024-12-31"),
  strategy = signal(
    entry  = cross_above(sma(close, 10), sma(close, 50)),
    exit   = cross_below(sma(close, 10), sma(close, 50)),
    symbol = "SPY",
  ),
  execution = execution(capital = 100_000),
  output    = output(results = "out/wf_fixed.json"),
)

walk_forward(base,
  train_window = "365d",
  test_window  = "90d",
  step_window  = "90d",
)
```

Use this when you already trust a parameter choice and you want to
see how it would have held up rolled forward, without re-fitting at
every window. Two things you still get for free vs. a single
backtest:

- **Stitched OOS curve**, identical in shape to the optimized form
  (badge reads `· OOS · N windows · fixed`).
- **Per-window OOS metrics table** — same columns minus IS Sharpe /
  Δ / PARAMS, since none of those apply.

When to reach for fixed instead of optimized:

- The strategy has no tunable hyperparameters worth sweeping
  (e.g. a hard-coded rule).
- You already have a winning parameter set from prior research and
  you're sanity-checking persistence, not re-deriving the choice.
- The sweep would be cost-prohibitive (`N_windows × M_cells`) but a
  single fixed run is fine.

The `best_params` field in the schema v5 output is `{}` for fixed
runs — that's the structural marker downstream consumers use to
distinguish fixed vs. optimized.

## What walk-forward proves

A single backtest tells you how a strategy did on a fixed series
with one parameter set. If you pick those parameters by running a
sweep on the same series and choosing the highest-Sharpe cell,
you've **overfit**: the sweep saw the data; the "winning"
parameters reflect noise as much as signal.

Walk-forward fixes this by structurally separating the data the
optimizer sees from the data it gets graded on. For every
window:

1. Train slice (the "IS" or "in-sample" window) — the sweep runs
   here. Pick the best params by `optimize.metric`.
2. Test slice (the "OOS" or "out-of-sample" window) — apply
   those exact params, run, report the result. **The optimizer
   never sees this segment.**
3. Roll the windows forward by `step_window` and repeat.

The stitched OOS equity curve is what your account would have
looked like if you'd retrained at every window boundary and
traded with the result. The IS-vs-OOS Sharpe delta is the
honesty check: large positive delta means the parameters worked
better in training than in real use, i.e. you were probably
fitting noise.

> **Walk-forward proves nothing about future returns.** It
> proves that *given this data*, the strategy and the optimizer
> together generalized inside the data window. It doesn't say
> anything about regime changes after the data ends.

## Reading the F4 BCKT panels

### IS vs OOS table

Each row is one window:

| Column | What to look at |
|---|---|
| IS SHARPE | What the sweep picked. By construction, this is a high-Sharpe cell — it's the best of the training-window cartesian product. |
| OOS SHARPE | What that choice produced on the held-out test slice. The number that actually matters. |
| Δ | `OOS - IS`. Strong negative Δ across many windows = overfitting. Mild negative is normal (peak IS Sharpe is always optimistic). |
| OOS RET | OOS total return on that test slice (sign-aware). |
| PARAMS | The best params chosen for this window. Useful next to PARAM DRIFT. |

### PARAM DRIFT mini-charts (sweep-optimized runs)

One thin line plot per axis, horizontal axis is window number,
vertical axis is the chosen value. Two patterns to recognize:

- **Flat-ish line.** Optimizer picks roughly the same value every
  window → the strategy is stable in this parameter (the IS
  surface has a clear peak that's not migrating).
- **Saw-tooth / noisy line.** Optimizer chases different cells
  every window → the IS surface is flat or multi-modal, the
  "best" param is dominated by noise. This is often a louder
  overfitting signal than a small IS-OOS Sharpe gap.

A high OOS Sharpe with a wandering PARAM DRIFT chart is a flag,
not a green light. The strategy may have worked despite the
optimizer being random.

## Window math gotchas

- **Calendar approximations.** `mo` = 30 days, `y` = 365 days.
  Walk-forward aligns by **bar index**, not wall-clock, so the
  approximation slop is bounded to "the rolling window is roughly
  this many bars" — fine for OOS validation. Don't expect month
  boundaries to snap to actual calendar months.
- **Partial tail windows are discarded.** If the data ends in the
  middle of a test slice, that segment is dropped — OOS metrics
  on a short tail are too noisy to be honest. If you see
  `N windows` but expected `N+1`, that's why; lengthen the data
  range or shorten `test_window`.
- **Train slice must have ≥ 2 bars.** Otherwise no indicator can
  warm up. Errors out at `compute_walk_forward_windows` rather
  than producing garbage.
- **Overlapping windows** (`step_window < test_window`) are
  supported. The stitched OOS curve skips duplicate timestamps
  from overlapping segments so the chart stays monotonic in
  time, but per-window IS/OOS metrics in the table count
  overlapping bars in both windows.

## Typical ratios

There's no canonical train/test ratio. Some rules of thumb that
work for daily-bar strategies:

- **3:1 IS:OOS** (`train_window = "1y"`, `test_window = "4mo"`) —
  conservative, large train slice gives the sweep enough room to
  produce a stable best cell. Use when the sweep grid is large
  (>50 cells).
- **1:1 IS:OOS** (`train_window = "180d"`, `test_window = "180d"`) —
  more windows out of the same data, useful when the strategy is
  fast (intraday, weekly rebalance) and you want statistical
  power.
- **Overlapping** (`step_window = test_window / 2`) — doubles the
  window count without changing IS/OOS sizes. Useful when data is
  short. Accept that per-window metrics aren't independent
  observations.

For sweep-optimized walk-forward, the cell count multiplies the
runtime: `N_windows × M_cells` backtests. 12 windows × 200 cells
= 2400 backtests; daily SPY at <10ms each = ~24 seconds. The
runner is single-threaded for now — concurrency is a follow-up.

## Search methods

`optimize = sweep(...)` defaults to grid search (full cartesian
product). Two other methods cut runtime when the parameter space
gets big:

- **`method = "random"`**: sample `n_samples` cells uniformly from
  the axes' joint distribution. Deterministic — same `seed`
  produces the same sample list. Useful when a 2-axis grid is
  fine but a 4-axis grid blows up cell count.

  ```qe
  optimize = sweep(base,
    axes      = [axis("fast", values = [3, 5, 8, 13, 21]),
                 axis("slow", values = [20, 30, 50, 80, 130]),
                 axis("commission_bps", values = [0.5, 1.0, 1.5, 2.0])],
    method    = "random",
    n_samples = 20,            # 20 of 100 cells per train slice
    seed      = 42,            # optional; default = stable hash of axes
    metric    = "sharpe",
  )
  ```

- **`method = "coarse_to_fine"`**: run the coarse grid (stage 1),
  pick the top-K winners by metric, then refine around each by
  interpolating midpoints between the winner and its axis
  neighbors (stage 2). Integer-only axes have refined midpoints
  rounded so window kwargs (`sma(close, fast)`) keep working.

  ```qe
  optimize = sweep(base,
    axes   = [axis("fast", values = [5, 10, 20]),
              axis("slow", values = [50, 100, 200])],
    method = "coarse_to_fine",
    levels = 2,                # 1 refinement pass after the coarse grid
    top_k  = 3,                # zoom around top 3 winners
    metric = "sharpe",
  )
  ```

  Approximately `coarse_grid + top_k × neighbor_count` train runs
  per window. `coarse_to_fine` is walk-forward-only — the F4 BCKT
  manual heatmap rejects it (the UX doesn't fit auto-refinement).

## Universe-based bases (`signalize_universe` + per-leg data)

A `base = backtest(...)` whose strategy is `signalize_universe(...)`
— or any `portfolio(...)` whose legs carry their own `data = ...`
templates — works in `walk_forward` end to end. Example:

```qe
let mom = factor("mom", expr = lag_return(close, 252))

let base = backtest(
  data = [],                                # ← legal: every leg supplies its own data
  strategy = signalize_universe(mom,
    universe = universe(
      symbols = ["XLK", "XLF", "XLE", "XLV", "XLY"],
      data    = yahoo_template("1d", "2018-01-01", "2024-12-31")),
    top_k    = 2,
    bottom_k = 0),                          # long-only rotation
  execution = execution(capital = 100_000),
  output    = output(results = "out/wf_universe.json"),
)

walk_forward(base,
  train_window = "365d",
  test_window  = "90d",
  step_window  = "90d",
)
```

What happens:

1. The runner loads every symbol in the universe via the per-leg
   data templates (one yahoo / file fetch per symbol). `base.data
   = []` is legal when every leg supplies its own data.
2. `align_intersection` runs once across all symbols; train + test
   windows are computed on the aligned bar grid.
3. Each window runs the whole universe through the engine (every
   leg sees that window's slice of data). OOS metrics are
   computed on the combined per-window equity curve, same as a
   `portfolio()` base.
4. `optimize = sweep(base, ...)` works on top: each train slice
   re-runs the sweep, picks the best params, applies them to the
   test slice.

This makes sector-rotation / long-only momentum strategies
walk-forward-testable without the user having to manually fan
out N copies of `signal(...)`.

## v1 limits

| | Status |
|---|---|
| Continuous-portfolio forward-test (each window keeps the previous balance) | Not supported. Every test slice starts from `execution.capital` fresh. Forward-test is its own runner (future). |
| Concurrent windows / cells | Single-threaded. Trivial concurrency available later (windows are independent). |
| Bayesian / Hyperband search inside each train slice | Out of scope — random + coarse-to-fine cover the common "shrink the grid" cases; Bayesian would mean ML deps the project doesn't need. |
| HTML report for walk-forward | Not yet — the JSON is there (schema v5) but `templates/report.html` only knows v3 / v4. F4 BCKT is the canonical view for now. |

## Schema v5 result shape

```jsonc
{
  "schema_version": 5,
  "strategy": {"name": "walk_forward",
               "params": {"train_ns":..., "test_ns":...,
                          "step_ns":..., "optimized": true|false}},
  "summary":         { ...OOS stitched... },
  "equity_curve":    [...OOS stitched...],
  "initial_capital": 100000.0,
  "walk_forward": {
    "n_windows": N,
    "windows": [
      { "train": {"start_ns":..., "end_ns":...},
        "test":  {"start_ns":..., "end_ns":...},
        "in_sample_metrics":     { ... },
        "out_of_sample_metrics": { ... },
        "best_params": {"strategy.fast": 15, ...},
        "oos_trades": [...] },
      ...
    ]
  }
}
```

`best_params` is `{}` for the fixed-param runner (no
`optimize`); populated for the sweep-optimized runner. Keys are
the axis paths from the `optimize.axes` array. The whole array
is sorted by axis name so the JSON round-trips deterministically.

## See also

- [`docs/qe-language.md`](qe-language.md) — `walk_forward(...)`
  reference in the language guide.
- [`docs/multi-strategy.md`](multi-strategy.md) — sibling feature;
  `walk_forward(base = backtest(strategy = portfolio(...)))` is
  supported.
