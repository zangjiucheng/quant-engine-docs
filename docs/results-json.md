# `results.json` schema reference

Stable typed reference for everything `qe_run` writes. Use this when
your Python aggregation script needs to know whether a field is
`max_drawdown` or `max_dd`, what `n_*` keys count which thing, and
which shape the equity curve will be in.

Three writers emit results.json with different top-level shapes —
they share the same per-cell / per-leg `summary` block schema:

| Shape | Writer | Top-level emitted by |
|---|---|---|
| Multi-asset run | `qe::io::write_multi_results_json` | `qe_run <backtest.qe>` |
| Multi-strategy portfolio run | `qe::io::write_portfolio_results_json` | `qe_run <backtest.qe>` whose `strategy = portfolio(...)` |
| Walk-forward run | `qe::io::write_walk_forward_results_json` | `qe_run <walk_forward.qe>` |
| Sweep aggregate | inline in `qe_run.cpp` | `qe_run <sweep.qe>` |

`factor_report.json` has its own [schema](factor-research.md);
its per-leg block uses the same key names as the `summary` block
below (`max_drawdown`, not `max_dd`).

## Canonical `summary` block

Every `summary` (top-level and per-leg) uses the SAME key set —
populated by `qe::analytics::summarize_equity_only()` so they can't
drift. NaN / inf values serialize as JSON `null`.

| Field | Type | Definition |
|---|---|---|
| `total_return` | number\|null | `equity[-1] / equity[0] - 1` (or `equity[-1] / capital - 1` when a `capital_override` applies — per-leg blocks use this against the leg's allocation) |
| `cagr` | number\|null | Annualized total return over the run's bar window |
| `sharpe` | number\|null | `mean(period_returns) / stddev(period_returns) * sqrt(periods_per_year)` |
| `sortino` | number\|null | Same as Sharpe but downside-only deviation. `+inf` when there's zero downside |
| `max_drawdown` | number | Worst peak-to-trough drawdown, positive fraction. `0.0` when monotonic non-decreasing |
| `win_rate` | number\|null | Fraction of round-trip trades with `pnl > 0`. `null` when no completed round-trips, or when the layer is multi-symbol / multi-leg (pairing isn't faithful) |
| `avg_win_loss` | number\|null | `mean(winning round-trips) / mean(losing round-trips)`. `null` when one side is empty |
| `n_trades` | integer ≥ 0 | Number of submitted trades on this layer |
| `n_bars` | integer ≥ 0 | Number of equity-curve points on this layer |

**Per-leg vs top-level**: `summary` inside `strategies[i]` carries
the same keys with the same semantics, scoped to that leg's equity
curve and trade list.

## Equity curve

By default (`output(format = "objects")` or omitted), `results.json`
contains:

```json
"equity_curve": [
  {"t": <int64 ns>, "equity": <double>},
  ...
]
```

With `output(format = "columnar")`, the same data is split
across two parallel arrays and `equity_curve` is omitted:

```json
"equity_ts":     [<int64 ns>, ...],
"equity_values": [<double>, ...]
```

`equity_ts.size() == equity_values.size()` is guaranteed; the
dashboard loader rejects a file that has both. Per-leg
`strategies[i]` follows the same format as the top-level — never
mixed.

Use `columnar` when:
- The run is long-horizon (10y daily ≈ 2500 points, 1y minute ≈
  100k points) and you parse the result in Python — `equity_ts` +
  `equity_values` is ~10× faster to deserialize than the dict-per-
  point shape.
- You want a JSON that's smaller on disk (no repeated `"t"` /
  `"equity"` keys).

Use `objects` (the default) when:
- The dashboard is your primary consumer and you have no reason to
  change.
- A downstream script you don't control depends on the legacy shape.

## Top-level versions

`schema_version` indicates the layout:

| Version | Writer | Notable additions |
|---|---|---|
| 1 | single-symbol `RunResult` | original |
| 2 | `MultiRunResult` without `bars` | `per_symbol[]` |
| 3 | `MultiRunResult` with `bars` | per-symbol Sharpe / Sortino / max_drawdown / win_rate; `per_sector[]` |
| 4 | `PortfolioRunResult` | `strategies[]` with per-leg `summary` + `equity_curve` / columnar pair |
| 5 | `WalkForwardRunResult` | `walk_forward.windows[]` with IS / OOS metrics per fold |

## `trades[]` per-fill fields (EPIC-77)

Every fill written under `trades[]` (and under each `strategies[i].trades`
in v4 portfolios, plus `walk_forward.windows[i].oos_trades` in v5)
carries the engine's view of the fill itself **and** the decision
context that produced it. Slippage and commission are emitted as
absolute dollar amounts (engine already converted from bp at fill
time); a downstream consumer can recover the bp-normalized slippage
from `(price - decision_price) / decision_price * 1e4` and break it
down by buy / sell side using the sign of `qty`.

| Field | Type | Description |
|---|---|---|
| `t` | int (ns) | Fill timestamp — the bar at which the engine executed the order. |
| `symbol_idx` / `symbol` | int / string | Per-symbol identifier (multi-symbol writers only). |
| `price` | number | Fill price (slippage already baked in via `effective_fill_price`). |
| `qty` | number | Signed quantity: positive = buy, negative = sell. |
| `commission` | number\|null | Commission paid on this leg, USD. |
| `slippage` | number\|null | Absolute slippage cost on this leg, USD. |
| `decision_t` | int (ns) | Bar timestamp at which the strategy emitted intent — the bar BEFORE this fill under `fill_model = "next_open"`. `0` when the trade was constructed without engine context (fixture-only). |
| `decision_price` | number\|null | Close price at the decision bar. `null` when no engine context. |

Sign convention for downstream slippage analysis:
`signed_slippage_bp = (qty > 0 ? +1 : -1) * (price - decision_price) / decision_price * 1e4`
— positive = unfavorable on either side.

`schema_version` is unchanged (the new fields are additive). Older
consumers can ignore `decision_t` / `decision_price` without
re-reading the schema; new consumers should treat the absence of
either field as "no decision context recorded for this fill" and
fall back to whatever reconstruction they used pre-EPIC-77.

## v4 portfolio correlation block (EPIC-78)

Every `qe_run <portfolio.qe>` output gains a top-level `correlation`
block computed from the per-leg equity curves at write time:

```json
"correlation": {
  "basis":  "daily_log_returns",
  "labels": ["lv", "mom", "carry"],
  "matrix": [
    [1.0, -0.15, 0.42],
    [-0.15, 1.0, 0.08],
    [0.42, 0.08, 1.0]
  ],
  "n_obs": 2543
}
```

| Field | Type | Description |
|---|---|---|
| `basis` | string | Always `"daily_log_returns"` in v1. Field exists so a future rolling / arithmetic-return variant can land without breaking JSON readers. |
| `labels` | string[] | Per-leg name, same order as `strategies[]`. `labels[i]` corresponds to `matrix[i][*]`. |
| `matrix` | number\|null[][] | NxN Pearson correlation matrix. Diagonal is exactly `1.0`; off-diagonal entries are `null` when the pair has fewer than 30 overlapping non-NaN return samples. Symmetric exactly. |
| `n_obs` | integer | Count of timestamps at which *every* leg's return is finite (the cross-strategy overlap, NOT per-pair). |

Sign convention follows the textbook: `+1.0` = perfectly co-moving,
`-1.0` = perfectly anti-correlated. Use for diversifier search by
inspecting `matrix[champion_idx][candidate_idx]` — values near `0`
indicate the candidate's returns are roughly independent of the
champion's.

`schema_version` stays `4` (additive top-level field). Older readers
that don't know about `correlation` keep working unchanged.
## `n_*` field meanings — at a glance

| Field | Layer | Counts |
|---|---|---|
| `summary.n_bars` | any | Equity-curve points on this layer |
| `summary.n_trades` | any | Submitted trades on this layer |
| `stats.n_cells` | sweep aggregate | Cells enumerated |
| `stats.n_ok` / `stats.n_failed` | sweep aggregate | Cell run-outcomes |
| `per_symbol[i].n_trades` | v2/v3 | Trades that hit this symbol |
| `per_sector[i].n_symbols` | v3 | Symbols mapped into this sector |

`factor_report.json` has `n_periods` (LS rebalance periods) — that's
a different unit from `n_bars` and intentionally stays distinct.

## See also

- `docs/factor-research.md` — `factor_report.json` schema.
- `docs/qe-language.md` — `output(...)` kwarg reference.
