# Multi-strategy portfolios

`portfolio(strategies=[...], weights=[...])` in `.qe` runs N
independent signals on the same data with a fixed slice of the
total capital each. The combined backtest stays a single value
(`backtest(strategy = portfolio(...))`), but the output carries
per-strategy attribution so you can see who contributed what.

This document explains the **semantics** — what the runner
actually does, why we chose this shape, and how to read the
attribution panel.

## TL;DR

```qe
let trend = signal(
  entry = cross_above(sma(close, 10), sma(close, 50)),
  exit  = cross_below(sma(close, 10), sma(close, 50)),
  symbol = "SPY",
)
let rev = signal(
  entry = rsi(close, 14) < 30,
  exit  = rsi(close, 14) > 70,
  symbol = "SPY",
)

backtest(
  data     = yahoo("SPY", "1d", "2020-01-01", "2024-12-31"),
  strategy = portfolio(
    strategies = [trend, rev],
    weights    = [0.6, 0.4],
    names      = ["trend", "rev"],
  ),
  execution = execution(capital = 100_000),
  output    = output(results = "out/portfolio.json"),
)
```

- Each child gets `capital × weight[i]` at start
  (60 000 / 40 000 above).
- Each child runs as an independent backtest on the same series.
- Combined equity at bar t = sum of every child's equity at bar t.
- F4 BCKT shows the combined curve in bold, each sub-strategy as
  a thin coloured overlay, and an `ATTRIBUTION` panel with
  per-strategy metrics.

## Execution model — N parallel backtests

The runner is `qe::backtest::run_portfolio` and it does exactly
one thing: for each child `SignalValue`, build an
`ExpressionStrategy`, allocate a fresh `MultiPortfolio` with
`capital × weight[i]`, run `MultiEngine::run` on the shared
series, stash the equity curve + trades. Then aggregate.

Three consequences worth knowing about:

1. **No cross-strategy netting.** If `trend` is long AAPL and
   `rev` is short AAPL, the combined book holds *both* positions
   simultaneously. They don't cancel to zero at the position
   level — they cancel at the equity level (their PnL changes
   offset). This is what "independent backtests" means: each
   child is making its own decisions about its own slice of
   capital.

2. **Per-strategy attribution is exact.** Each child has its own
   equity curve and its own trades, so `ATTRIBUTION`'s
   per-strategy Sharpe / Return / Max DD numbers are real — they
   describe what that child did with its own sub-capital, not a
   share-of-portfolio estimate.

3. **Hot loop is unchanged.** The engine doesn't know `portfolio`
   exists; it runs `MultiEngine::run<ExpressionStrategy>` N times
   in sequence, exactly the same code path single-signal uses.

## What ships today vs what's still future

| Feature | Status |
|---|---|
| Within-run cross-strategy rebalancing (`rebalance = "monthly"`, etc.) | Planned. Needs cross-child cash flow that doesn't fit the "N independent runs" shape. Current code enforces `rebalance = "never"`. |
| Risk parity / inverse-vol / dynamic weights | Future. Weights are constants today. |
| Strategy correlations / portfolio variance optimization | Future. Engine reports correlations as a side effect of running the children; no optimization layer. |
| Each strategy on its own data universe | ✅ **Shipped**. Each `signal(...)` child can carry its own `data = file(...) \| yahoo(...) \| csv_url(...)` template; the runner appends per-leg series to the shared `MultiBarSeries` after loading any top-level `backtest.data = [...]` entries. `backtest.data = []` is legal when every leg supplies its own data. |
| Nested portfolios — `portfolio(strategies = [signalize_universe(...), signal(...)])` | ✅ **Shipped**. The parent flattens any nested `PortfolioValue` (anything that returns one — `signalize_universe`, `portfolio` inside `portfolio`) into its own leg list. Per-slot weights distribute across inner legs proportional to inner weights: a single universe slot at weight 1.0 splits 1/N across N legs; mixing `signal(...)` and `signalize_universe(N legs)` at 50/50 gives the bare signal 50% and each universe leg `0.5/N`. Names are auto-prefixed `${slot}_${inner_symbol}` so the flat list stays distinguishable. |
| Portfolio inside `walk_forward(...)` | ✅ **Shipped**. `walk_forward(base = backtest(strategy = portfolio(...)))` runs each child on every train + test slice, combines the equity curves, and reports IS / OOS metrics on the combined book. Works for `signalize_universe`-shaped bases too. |
| Portfolio inside `sweep(...)` | ✅ **Shipped**. Same caveat — works for any portfolio-shaped strategy including `signalize_universe`. |

## Dashboard behaviour with flat / flattened books

The dashboard reads the **flat** `portfolio_strategies[]` array
from `results.json`. After portfolio flattening (nested
`signalize_universe(...)` etc.) the array holds one entry per
flat leg, so:

- **ATTRIBUTION table** renders one row per leg. For a
  `signalize_universe(...)` over 9 sector ETFs, expect 9 rows;
  the table is scrollable so larger universes (S&P 500, etc.)
  display without truncation.
- **Equity-curve overlay** draws a thin semi-transparent line
  per leg using an 8-colour palette that cycles when N > 8. The
  bold combined line sits on top.
- **Title chip** shows `portfolio · N` where N is the flat leg
  count, not the slot count.

There is no separate "slot view" in the UI today — the flat list
is what the engine sees, so the dashboard mirrors it. Users who
want to verify the slot → leg mapping should read the `name`
column in ATTRIBUTION (auto-prefixed `${slot}_${inner_symbol}`).

## Reading the ATTRIBUTION panel

When F4 BCKT detects a schema v4 `results.json`, the rightmost
panel switches from the monthly heatmap to `ATTRIBUTION`:

| Column | Meaning |
|---|---|
| STRATEGY | `names[i]` from the `portfolio(...)` call, or auto-`strategy_N` if omitted. |
| WEIGHT | `weights[i]`, as a percentage. |
| RETURN | This child's total return on its own sub-capital. Compare to the combined Return in KEY STATS. |
| SHARPE | This child's Sharpe — same caveat as any sample-Sharpe: short series → noisy. |
| MAXDD | Max drawdown observed on this child's curve. |
| CONTRIB | `child_pnl / combined_pnl`, signed. Sums to 100% when combined PnL is non-zero. Negative when this child *lost* money in a winning portfolio (or vice versa) — useful for spotting "this strategy is dragging the book". |

Reading example: combined Return +5%, `trend` Contrib +120%,
`rev` Contrib −20% → `rev` lost money but `trend` carried the
portfolio. That's a signal to question whether `rev` is worth its
weight at all.

## When to use portfolios

The most common practical case is **uncorrelated strategies on
the same instrument** — trend-following + mean-reversion on SPY,
say. Each strategy has different drawdown profiles, and combining
them with fixed weights gives a smoother combined curve than
either alone (assuming the strategies are actually
uncorrelated — verify with the equity overlay; if both children
move together, you don't have a diversification benefit, you have
a leverage problem).

Less common but also reasonable: **the same strategy with
different parameter sets**, e.g. `signal(... sma(close, 10/50))`
+ `signal(... sma(close, 20/100))`, to hedge against a single
parameter choice doing badly out-of-sample. This is the
manually-staged version of what `walk_forward(... optimize)` does
automatically — see [`docs/walk-forward.md`](walk-forward.md).

## Schema v4 result shape

`results.json` for a portfolio run is schema_version 4. Top-level
`summary` and `equity_curve` are the combined view; `strategies[]`
carries the per-child data:

```jsonc
{
  "schema_version": 4,
  "strategy": {"name": "portfolio",
               "params": {"names": [...], "weights": [...]}},
  "summary": { ...combined... },
  "equity_curve": [...combined...],
  "initial_capital": 100000.0,
  "strategies": [
    { "name": "trend",
      "capital": 60000.0,
      "weight": 0.6,
      "summary": { ...per-child... },
      "equity_curve": [...],
      "trades": [...]
    }, ...
  ]
}
```

The HTML report renders the same shape — combined equity bold +
sub-strategy lines overlaid + a strategy attribution table after
the KPI cards.

## See also

- [`docs/qe-language.md`](qe-language.md) — `portfolio(...)`
  reference in the language guide.
- [`docs/walk-forward.md`](walk-forward.md) — sibling feature for
  out-of-sample validation.
