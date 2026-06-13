# Forecasting layer

> This page covers the rolling-ridge and rolling-lasso predictors
> (v1 + the v2 kwarg-form extensions from EPIC-50a and EPIC-50b).
> A second generation with tree / NN predictors is planned
> (EPIC-51) and not yet shipped.

Rolling-fit linear predictors (`RollingRidge`, `RollingLasso`)
exposed to `.qe` configs as `predict_return(...)`. The motivation
is to start data-driven strategy research from a known baseline —
ridge / lasso regression — that any later ML approach has to beat
to justify its complexity.

The function has two parse forms that produce the same machinery:

- **v1 positional** — `predict_return(lookback, alpha, f1, ...)`.
  Backward-compatible with every fixture in the repo. Forecasts
  next-bar close-to-close return, ridge-solved.
- **v2 kwarg** (EPIC-50a + EPIC-50b) — `predict_return(lookback =
  ..., features = [...], alpha = ... | lambda = ..., horizon = H,
  target = E, method = "ridge" | "lasso")`. Unlocks four extensions:
    * `horizon = H` — forecast `H` bars ahead instead of one.
    * `target = E` — replace the hardcoded close-to-close return
      with any signal-context expression (e.g. volatility,
      multi-bar return, sign of return).
    * `method = "lasso"` + `lambda = L` — switch the per-call-site
      solver to a coordinate-descent L1-regularized fit. Same span-
      taking API, sparse coefficients, sklearn-matching to 1e-6.
    * `method = "ridge"` (the default) keeps existing fixtures and
      walk-forward results bit-for-bit identical.

The two forms cannot be mixed in a single call — pick one. v1
calls stay bit-for-bit identical; the binder asserts
`horizon = 1` kwarg ≡ v1 positional on the same features.

## Quick start

```qe
backtest(
  data = yahoo("SPY", "1d", "2018-01-01", "2024-12-31"),
  strategy = signal(
    entry = predict_return(
              252, 1.0,
              lag_return(close, 1),
              lag_return(close, 5),
              rolling_zscore(close, 20),
              rsi(close, 14) / 100.0
            ) > 0.001,
    exit  = predict_return(
              252, 1.0,
              lag_return(close, 1),
              lag_return(close, 5),
              rolling_zscore(close, 20),
              rsi(close, 14) / 100.0
            ) < 0,
    symbol = "SPY",
  ),
  execution = execution(capital = 100_000),
  output    = output(results = "out/forecast.json"),
)
```

What this does, per bar:

1. Compute the four feature values from the bar's `close`.
2. If a prior bar's features are stashed, push
   `(features_{t-1}, return_t)` into the ridge — keeping the training
   pair causal — and refit.
3. Predict `return_{t+1}` from this bar's features.
4. Compare the prediction to the entry / exit thresholds, which
   becomes the strategy's signal for the next bar's fill.

The first `lookback + feature_warmup` bars produce NaN forecasts; the
strategy stays flat through warm-up.

## API

### `predict_return(lookback, alpha, f1, f2, ..., fN)` — v1 positional

| Arg          | Type          | Constraints                          |
|--------------|---------------|--------------------------------------|
| `lookback`   | int literal   | `> 0`, `>= n_features + 1`           |
| `alpha`      | double literal| `>= 0` (use `0` for plain OLS)       |
| `f1..fN`     | signal expr   | at least one; up to 64 in v1         |

Returns a `double` — the next-bar forecast. Warm-up returns NaN; any
feature being NaN this bar also returns NaN and breaks the causal
training chain for one bar (better than corrupting the ring with
garbage). Each call site allocates an independent `RollingRidge`
instance; an `entry` and `exit` that both call `predict_return(...)`
do not share state in v1 — they train in parallel on the same data
and converge to similar coefficients.

### `predict_return(lookback = ..., features = [...], method = ..., ...)` — kwarg form (EPIC-50a + EPIC-50b)

The kwarg form unlocks four extensions and accepts every original
v1 feature unchanged:

| Kwarg      | Type             | Default      | Constraint                          |
|------------|------------------|--------------|-------------------------------------|
| `lookback` | int literal      | — required   | `>= n_features + 1`                 |
| `features` | array literal    | — required   | `[expr, expr, ...]`, non-empty      |
| `method`   | string literal   | `"ridge"`    | `"ridge"` or `"lasso"`              |
| `alpha`    | double literal   | required if `method = "ridge"`; rejected for lasso | `>= 0` |
| `lambda`   | double literal   | required if `method = "lasso"`; rejected for ridge | `> 0` |
| `horizon`  | int literal      | `1`          | `>= 1`                              |
| `target`   | signal-context expression | next-bar close-to-close return | causal (backward-looking) |

`method` is the only signal-context kwarg that accepts a string
literal — strings remain rejected everywhere else in the signal
layer. The binder gives a targeted error if a string sneaks into
any other position.

**Horizon (`horizon = H`)** — forecast `H` bars ahead instead of one.
At bar `t`, the predictor returns its forecast for `target_{t+H}`.
Training pairs are `(features_{t-H}, target_t)` — the causal lag
matches the forecast horizon, so a trace recorded with `horizon = 5`
pairs each `y_hat` with the realized target observed exactly five
bars later. Warm-up grows by `H - 1` bars (the ring must accumulate
`H` past feature vectors before the first training pair can push).

**Target (`target = <expr>`)** — replace the v1 default
(`(close - prev_close) / prev_close`) with any signal-context
expression. Backward-looking forms (`lag_return`, `rolling_return`,
`rolling_zscore`, etc.) are causal by construction. Try volatility
prediction with `target = rolling_vol(close, 20)`, or longer-horizon
return prediction with `target = lag_return(close, 5)` paired with
`horizon = 5`.

```qe
# Predict the 5-day-ahead close-to-close return from a few
# rolling features. Output is causal at bar t for bar t+5.
predict_return(
  lookback = 100,
  alpha    = 1.0,
  horizon  = 5,
  features = [
    lag_return(close, 1),
    rolling_zscore(close, 20),
    rsi(close, 14),
  ],
  target = lag_return(close, 5),
)
```

**Method (`method = "ridge" | "lasso"`)** — pick the solver. Ridge
(the default) uses the closed-form `(XᵀX + αI′)⁻¹Xᵀy` via Cholesky.
Lasso uses coordinate descent on the centered design matrix and
matches `sklearn.linear_model.Lasso(alpha = λ, fit_intercept = True)`
to ~1e-6 on the canonical 100×5 test design. The crucial behavioral
difference is **sparsity**: at moderate `lambda`, lasso pushes
irrelevant feature coefficients to exact zero, which both removes
noise from the forecast and makes the learned model trivially
inspectable. Use lasso when the feature set is wide or includes
candidates you suspect are noise; use ridge when every feature has
prior justification and you mostly want shrinkage.

```qe
# Same features, lasso-solved with a tighter penalty. Coefficients
# of features that aren't pulling weight collapse to exact zero.
predict_return(
  lookback = 252,
  method   = "lasso",
  lambda   = 0.01,
  features = [
    lag_return(close, 1),
    lag_return(close, 5),
    rolling_zscore(close, 20),
    rsi(close, 14) / 100.0,
  ],
)
```

`alpha` and `lambda` are mutually exclusive — passing both, or
passing `lambda` with `method = "ridge"` (or `alpha` with `method =
"lasso"`), is rejected at bind time. `lambda > 0` is required for
lasso; for pure OLS use `method = "ridge", alpha = 0` (which falls
back to the Tikhonov-perturbed solve).

The new fields on `ForecastTracePoint` carry the horizon:

```cpp
struct ForecastTracePoint {
    std::int64_t ts_ns;
    double       bar_index;     // bar at which `y_realized` was observed
    double       y_hat;         // forecast emitted `horizon` bars earlier
    double       y_realized;    // observed value of `target` at this bar
    std::size_t  horizon;       // == 1 in v1 traces; H in v2 traces
};
```

### Feature primitives

| Function                   | Output                                     |
|----------------------------|--------------------------------------------|
| `lag_return(price, k)`     | `(p_t - p_{t-k}) / p_{t-k}`; NaN k bars    |
| `rolling_vol(x, n)`        | n-window sample std; NaN n-1 bars          |
| `rolling_zscore(x, n)`     | `(x_t - mean_n) / std_n`; 0 on constants   |

All are per-call-site stateful, O(1) per push for `lag_return`, O(n)
for the rolling stats — well under the per-bar budget for any
reasonable window.

## Walk-forward validation

The forecasting layer composes with the `walk_forward(...)`
harness. Wrap a `backtest(...)` that uses `predict_return` in
walk-forward windows; each test slice reports OOS metrics that are
**not** in the predictor's training set:

```qe
walk_forward(
  base = backtest(
    data = yahoo("SPY", "1d", "2018-01-01", "2024-12-31"),
    strategy = signal(
      entry = predict_return(60, 1.0, lag_return(close, 1)) > 0.001,
      exit  = predict_return(60, 1.0, lag_return(close, 1)) < 0.0,
      symbol = "SPY",
    ),
    execution = execution(capital = 100_000),
  ),
  train_window = "365d",
  test_window  = "90d",
  step_window  = "90d",
)
```

Read the resulting `results.json`'s `walk_forward.oos_metrics`: if
in-sample sharpe is high but OOS sharpe sits near zero, the predictor
is fitting noise. This is the canonical overfit smell-test the
forecasting layer is designed to support.

## Inspecting forecasts in the dashboard

Opt in by setting `record_forecast = true` on `execution(...)`:

```qe
execution(
  capital         = 100_000,
  commission_bps  = 1,
  record_forecast = true,
)
```

`qe_run` then drains the per-bar (`y_hat`, `y_realized`) trace from
every `predict_return(...)` call site after the run and appends a
`forecasts[]` block to `results.json`:

```json
"forecasts": [
  {
    "id":         "entry#0",
    "lookback":   60,
    "alpha":      1.0,
    "n_features": 4,
    "metrics": {
      "rmse":                 0.0142,
      "directional_accuracy": 0.51,
      "r2_vs_naive":          0.03,
      "n":                    243
    },
    "series": [
      { "ts_ns": ..., "bar_index": ..., "y_hat": ..., "y_realized": ... },
      ...
    ]
  }
]
```

The F4 BCKT screen's bottom-right slot detects the block and switches
into a scatter view: `y_hat` on x, `y_realized` on y, with a 45°
identity line and a y=0 reference. The header badge restates
`n / directional accuracy / RMSE / R² vs naïve`. Points above the
identity line are bars where the model under-predicted the realized
return; the top-right and bottom-left quadrants are direction-correct.

Cost note: each trace point is 24 bytes on disk and a few hundred
bytes per render. For 60-bar smoke runs this is free; for 1M-bar
minute runs you'll add ~25 MB to `results.json`. Leave the flag off
unless you actually want the overlay.

Limitations of the v1 overlay:
- Only the first call site is rendered (typical use: `entry#0` and
  `exit#0` are identical predictors, so the picture is the same). A
  selector across call sites is a follow-up.
- `walk_forward(...)` doesn't currently drain predictor traces
  through its window-stitched runner — the overlay shows up on
  single-pass backtests only.

## Limitations

- **Targets**: `target = ...` (EPIC-50a) lets you predict
  volatility, multi-bar return, or any backward-looking expression.
  Forecast-of-classification (direction probability with a softmax
  head) still requires a tree / NN predictor (EPIC-51).
- **Methods**: ridge (default) and lasso (EPIC-50b) ship today.
  Bayesian regression and tree-based models (XGBoost, LightGBM) are
  planned as separate `predict_*` builtins under EPIC-51.
- **No cross-sectional models**. The predictor takes per-bar features
  from one symbol; cross-asset (e.g. residualize on a market factor)
  needs a future multi-symbol predict primitive.
- **No shared state across signal positions**. An `entry` and `exit`
  expression that both call `predict_return(...)` instantiate
  independent ridges. Practical impact: ~2× compute for the same fit;
  no correctness issue.
- **No live coefficient inspection from the dashboard yet**. The F4
  BCKT screen renders the OOS curve when walk-forward is used; an
  overlay panel showing forecast-vs-realized is a follow-up.

## Implementation pointers

| Concern                    | File                                       |
|----------------------------|--------------------------------------------|
| Closed-form ridge solver   | `include/qe/forecast/rolling_ridge.hpp`    |
| Coordinate-descent lasso   | `include/qe/forecast/rolling_lasso.hpp`    |
| Feature indicators         | `include/qe/indicators/lag_return.hpp`<br>`include/qe/indicators/rolling_stats.hpp` |
| DSL builtin registration   | `src/dsl/env.cpp`                          |
| Per-bar dispatch           | `src/dsl/evaluator.cpp` (case `PredictReturn`) |
| Warmup math                | `src/dsl/analysis.cpp`                     |
| End-to-end smoke           | `tests/fixtures/forecast_smoke.qe`         |

## Pitfalls

- **`lookback` too small**: with 5 features + intercept = 6
  coefficients, `lookback = 6` is the minimum but the design matrix
  will be rank-deficient on any colinear inputs; bump to at least
  `4 × n_features` for sane fits. The binder rejects
  `lookback < n_features + 1` to catch the obviously broken case.
- **Highly correlated features**: the ridge is robust against
  collinearity at the LDLT level, but the learned slopes become
  noisy. Prefer a small set of orthogonal-ish features
  (`lag_return` at different horizons, a z-score, RSI normalized).
- **Refit cost on minute bars**: at lookback 1000 and 5 features
  `RollingRidge::fit()` runs in ~250 µs. For 1M bars that's about
  4 minutes wall time of pure refit — acceptable for one-shot
  research, painful inside a sweep × walk-forward grid. Drop the
  lookback or thin the features when that bites.
- **Lasso convergence**: defaults are `max_iter = 200`, `tol = 1e-6`.
  For most signal-discovery tasks that's fine — once the ring
  slides by one sample, CD warm-starts from the previous fit and
  converges in 1–3 sweeps. If you're chasing tight numerical
  agreement with sklearn, the unit tests use `max_iter = 200_000,
  tol = 1e-12`; those knobs aren't exposed in the DSL because the
  signal layer doesn't need that precision.
- **Don't mix `alpha` and `lambda`**: the binder rejects passing
  both, or passing `lambda` with `method = "ridge"`, because they
  parameterize different objectives. The error message points to
  the offending kwarg.
