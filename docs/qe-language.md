# qe — config language (`.qe`)

The format `qe_run`, `qe_factor`, and the dashboard expect for
hand-authored backtest, sweep, walk-forward, and factor-research
configs.

A `.qe` file evaluates to **one value** — a `backtest(...)`,
`sweep(...)`, `walk_forward(...)`, or `research(...)`. The value's
variant decides which runner consumes it; there is no imperative
top-level, no side effects, no I/O at evaluation time.

## Top-level values

| Top-level value     | Runner                          | Dashboard panel | Reference                                       |
|---------------------|---------------------------------|------------------|-------------------------------------------------|
| `backtest(...)`     | `qe_run <file>.qe`              | **F4 BCKT**      | This page → §`backtest`                         |
| `sweep(...)`        | `qe_run <file>.qe`              | **F4 BCKT** sweep | This page → §`sweep`                            |
| `walk_forward(...)` | `qe_run <file>.qe`              | **F4 BCKT** walk-forward | [`walk-forward.md`](walk-forward.md)    |
| `research(...)`     | `qe_factor <file>.qe`           | **F7 FCTR**      | [`factor-research.md`](factor-research.md)      |
| `live(...)`         | `qe_daemon start <file>.qe`     | **F6 TRADE**     | [`live-trading-runbook.md`](live-trading-runbook.md) + this page → §`live` |

The dashboard's F3 WKSP screen detects the variant on **Cmd+S** and
dispatches to the matching CLI automatically. New to the project?
Start with [`first-factor-tutorial.md`](first-factor-tutorial.md)
for the WKSP → Cmd+S → F7 round trip.

## Why a language

Three things the old JSON shape couldn't handle without becoming ugly:

1. **DSL inside strings.** Strategy expressions like
   `cross_above(sma(close, 10), sma(close, 50))` used to live inside
   JSON string literals — no syntax help, no error location, awkward
   to write.
2. **Opaque `strategy.params`.** Every strategy invented its own
   key/value shape; the JSON loader couldn't enforce or discover it.
3. **Sweep boilerplate.** Sweep specs were a JSON copy-paste of a
   base backtest plus dotted-path strings like
   `"strategy.params.fast_window"` edited blind.

In `.qe`, the signal expression IS the strategy, let-bindings remove
the boilerplate, and the parser knows what's a number / string /
range so type errors surface with a column number.

## A first file

```qe
# example_ma_spy.qe — Cmd+S in F3 WKSP runs it.
let fast = 10
let slow = 50

backtest(
  data = yahoo("SPY", "1d", "2024-01-01", "2025-01-01"),
  strategy = signal(
    entry  = cross_above(sma(close, fast), sma(close, slow)),
    exit   = cross_below(sma(close, fast), sma(close, slow)),
    symbol = "SPY",
  ),
  execution = execution(
    capital        = 100_000,
    commission_bps = 5,
    slippage_bps   = 2,
    fill_model     = "next_open",
  ),
  output = output(results = "out/example_ma_spy.json"),
)
```

The file's last expression is what gets executed. Everything above
that expression is `let` bindings that get inlined when referenced.

## Grammar (EBNF)

```
file        = { let_binding } , expression ;
let_binding = "let" , ident , "=" , expression ;

expression  = or_expr ;

or_expr     = and_expr  , { "or"  , and_expr } ;
and_expr    = not_expr  , { "and" , not_expr } ;
not_expr    = [ "not" ] , compare ;

compare     = sum , [ cmp_op , sum ]
            | sum , ".." , sum , [ "step" , sum ] ;   (* range *)
cmp_op      = ">" | ">=" | "<" | "<=" | "==" | "!=" ;

sum         = product , { ( "+" | "-" ) , product } ;
product     = unary   , { ( "*" | "/" ) , unary } ;
unary       = [ "-" ] , primary ;

primary     = number
            | string
            | bool_lit
            | ident
            | call
            | array
            | "(" , expression , ")"
            ;

call        = ident , "(" , [ arg , { "," , arg } , [ "," ] ] , ")" ;
arg         = expression | ident , "=" , expression ;   (* kwarg *)

array       = "[" , [ expression , { "," , expression } , [ "," ] ] , "]" ;

bool_lit    = "true" | "false" ;
string      = '"' , { any-char-except-quote } , '"' ;   (* no escapes yet *)
ident       = letter , { letter | digit | "_" } ;
number      = digit , { digit | "_" } ,
              [ "." , digit , { digit | "_" } ] ;
letter      = "A".."Z" | "a".."z" | "_" ;
digit       = "0".."9" ;
```

- Whitespace and line comments (`# ...` to end-of-line) are skipped.
- Underscores in numeric literals are visual grouping only: `100_000`
  is `100000`. Trailing or leading underscores are rejected.
- Strings have no escape sequences in v1 — `\` is taken literally.
  Paths with `\` need to use that backslash as-is or live on a
  POSIX-style path; double-quote inside a string is not representable
  yet.

## Precedence (low → high)

```
or                       — left assoc
and                      — left assoc
not                      — prefix unary
==  !=  <  <=  >  >=     — non-assoc, single comparison only
a..b [step c]            — non-assoc, mutually exclusive with comparison
+  -                     — left assoc (binary)
*  /                     — left assoc
unary -                  — prefix
call / array / paren     — highest
```

Comparison and range are mutually exclusive at the same level —
`a..b < c` is a parse error. Use parentheses if you really mean one
of them: `(a..b) < c` is still meaningless (range isn't comparable),
but it's at least an evaluator error rather than a parse one.

## Two contexts: **config** vs **signal**

Every `.qe` value lives in one of two evaluation contexts:

| Context | Where it shows up | What's allowed |
|---|---|---|
| **config** | Top-level + every arg position except `signal(entry=…)` / `signal(exit=…)` | Let bindings, kwarg calls, ranges, arrays, strings, arithmetic, the config builtins below |
| **signal** | The `entry` / `exit` args of `signal(...)` | Per-bar variables (`close`, `open`, `high`, `low`, `volume`, `bar_index`); signal-layer functions; arithmetic / comparison / logical ops; positional args only — no kwargs, no let, no arrays, no strings |

The config evaluator slices each signal subtree out of the source AST,
inlines let-bound scalars, then hands the result to the existing
per-bar binder. A let bound to a signal sub-expression
(`let ma_fast = sma(close, 10)`) is also inlined where referenced
inside a signal context.

## Config builtins

### `backtest(data, strategy, execution?, output?, date_range?)`

Required:
- `data` — one `DataSpec` (`yahoo(...)` / `file(...)`) or an array
  of `DataSpec`s (multi-asset).
- `strategy` — a `signal(...)` value.

Optional:
- `execution` — defaults to `execution()` (zero fees, NextOpen,
  100k capital).
- `output` — defaults to no writes.
- `date_range` — slice the loaded data after fetch.

### `signal(entry, exit, symbol?, size?, data?)`

Required keyword args:
- `entry`, `exit` — signal-context expressions returning boolean.

Optional:
- `symbol` — target symbol; defaults to `"X"` (matches legacy
  single-symbol setups).
- `data` — **per-leg data source**. Defaults to "inherit
  `backtest.data`". When set inside a `portfolio(...)` leg, the
  runner loads the spec under the leg's `symbol` and appends it to
  the shared series before the portfolio loop, so each leg can pull
  from its own Yahoo fetch / CSV / URL. Requires an explicit
  non-default `symbol`. The leg's symbol must not collide with any
  symbol already in `backtest.data` — remove the conflict or drop
  the per-leg spec. Accepts any `DataSpec`-producing builtin:
  `yahoo(...)`, `file(...)`, or `csv_url(...)`. When every leg
  supplies its own `data`, `backtest.data = []` is acceptable.
- `size` — per-trade order quantity. **Either** a positive scalar
  literal (`size = 2.5`, the legacy form) **or** a signal-context
  expression that evaluates per bar at entry time. Examples:

  ```qe
  signal(
    entry = cross_above(sma(close, 10), sma(close, 50)),
    exit  = cross_below(sma(close, 10), sma(close, 50)),
    size  = 0.02 / rolling_vol(close, 20),  // vol-target sizing
  )
  ```

  Expression results that are `NaN` (warmup), `0`, or negative
  **skip the trade** — same convention as `entry` / `exit`
  evaluating false. Each entry is also defensively capped at 100 %
  of current equity at the most recent mark price; orders that
  would exceed it are scaled down. Compounding (`compound = true`)
  is suppressed when `size` is an expression — the user owns the
  equity-scaling math inside the expression.

### `yahoo(symbol, resolution, start, end?, cache_dir?)`

Fetches from the Yahoo Finance v7 endpoint. `symbol` and `start` are
required; `resolution` defaults to `"1d"` (also accepts `"1h"`,
`"1m"`, `"1wk"`, `"1mo"`). All five can be positional or keyword.

### `file(path, format?)`

Local CSV. `format` defaults to `"yahoo"` (the columnar layout
yfinance writes).

### `csv_url(url, format?, cache_dir?)`

Remote CSV. Fetched synchronously via HTTP once and cached by
content-hashed filename under `${HOME}/.qe_cache/csv_url/` (override
with `cache_dir`). Subsequent runs read the cached file directly;
delete it to force a re-fetch. `format` defaults to `"yahoo"`. Only
`http://` and `https://` are accepted — `file(...)` covers local
paths.

```qe
backtest(
  data = csv_url(
    "https://example.com/historical/SPY.csv",
  ),
  strategy = ...,
)
```

`csv_url(...)` is not yet usable as a `universe(...)` template —
cross-symbol research still needs `yahoo_template(...)` or
`file(...)` with a `%s` placeholder.

### `ibkr_historical(symbol, ...)`

**Not implemented.** Errors at parse time with a clear message. The
IBKR broker hook is currently live-trading only; for backtest data
use `file(...)`, `yahoo(...)`, or `csv_url(...)`.

### `execution(capital?, commission_bps?, slippage_bps?, fill_model?, compound?, impact_bps_per_pct_adv?)`

All keyword, all optional. Defaults: `capital = 100_000`, others
zero, `fill_model = "next_open"` (also accepts `"next_close"`),
`compound = false`, `impact_bps_per_pct_adv = 0`.

**`commission_bps`** accepts either a scalar (legacy form) or a
`maker_taker(...)` split. With the current `NextOpen` and
`NextClose` fill models all fills cross the spread → the engine
charges the `taker` rate. The `maker` arm is forward-compatible
config that starts mattering when a limit-touch fill model lands.

```qe
execution(
  commission_bps = maker_taker(maker = 0.0, taker = 5.0),
)
```

**`slippage_bps`** accepts either a scalar (every symbol pays the
same bps) or a `per_symbol(...)` map. The map form is
expressed as either positional `(symbol, bps)` pairs or
identifier-keyed kwargs. Symbols absent from the map pay 0 bps —
the scalar value is **not** used as a fallback.

```qe
execution(
  slippage_bps = per_symbol(SPY = 1.5, QQQ = 3.0, IWM = 4.5),
)

execution(
  slippage_bps = per_symbol("BRK.B", 2.0, "AAPL", 1.0),  # positional form
                                                         # for tickers with
                                                         # punctuation
)
```

**`impact_bps_per_pct_adv`** opts into a first-cut linear impact
model: each fill adds `impact_bps_per_pct_adv × (100 × |qty| /
ADV)` bps of slippage on top of the per-symbol / scalar base, where
ADV is the trailing 20-bar average volume up to (and not
including) the signal bar. Set to `> 0` to enable; default `0`
keeps every fill at the literal slippage.

`compound = true` rescales each new entry's submitted share count by
`current_equity / initial_capital` — gains from prior round-trips
enlarge subsequent entries, so winning strategies compound across
trades rather than realizing flat dollar gains. The matching exit
closes the exact open position, not the literal `size`. Within a
single held position the share count does not resize — use an
expression-form `size = ...` if you need that.

### `assert(condition, message)`

Pre-flight check evaluated at config-load time, before any data
fetch or engine run. Use as a top-level safety gate for
deployment configs. All three of these forms work and fire the
assert eagerly:

```qe
# Bare top-level — the natural form.
assert(n_symbols >= 5, "universe must have at least 5 symbols")
backtest(...)

# Underscore binding — the explicit side-effect form.
let _ = assert(n_symbols >= 5, "universe too small")
backtest(...)

# Named let — eager because the RHS is an assert call.
let _ok = assert(n_symbols >= 5, "universe too small")
backtest(...)
```

- `condition` — any expression that evaluates to a number or bool
  in config context. Comparison operators return `1.0` / `0.0`;
  arithmetic, let-bound scalars, and string equality all work.
- `message` — required string; surfaced verbatim in the resulting
  `Error{InvalidConfig, "assert: <message>"}`. Silent assertions
  in deploy bundles are worse than no assertions at all, so the
  message can't be omitted.

How the three forms work:
- The parser desugars bare top-level `assert(...)` into `let _ =
  assert(...)` automatically.
- `let _ = ...` (any RHS) is always eager — the underscore is the
  "side-effect-only" binding form.
- `let NAME = assert(...)` (any NAME) is also eager — the
  evaluator detects the assert call and fires it regardless of
  whether NAME is later referenced.

`condition == 0`, `false`, or `NaN` all fail. The returned value
is `1.0` on success — useful when chaining.

Field access on universe / factor values is a future task; for
now, materialize the values you want to assert via `let` first:

```qe
let n_symbols = 9        # known from your data fixture
assert(n_symbols >= 5, "universe too small")
```

### `output(results?, report?, format?)`

All optional, all strings. Empty / missing results / report means
"don't write".

`results` and `report` strings undergo `${ENV_VAR}` substitution
at load time so `qe_daemon` configs can declare stable output
paths independent of the current working directory:

```qe
output(results = "${HOME}/qe_runs/today.json")
```

Unknown variables stay literal — broken configs surface the unset
variable name in the resulting path rather than silently producing
something the daemon can't write to.

- `format` — `"objects"` (default) or `"columnar"`. Controls the
  equity-curve serialization shape in `results.json`:
  - `"objects"` (legacy): `equity_curve: [{"t": ..., "equity":
    ...}, ...]`
  - `"columnar"`: two parallel arrays `equity_ts: [int64]` +
    `equity_values: [double]` and no `equity_curve` key. ~10×
    faster to deserialize in Python on long-horizon backtests
    (10y daily, 1y minute). Per-leg `strategies[i]` follows the
    same shape. See `docs/results-json.md` for the full schema.

### `date_range(start?, end?)`

ISO `YYYY-MM-DD` strings; either side can be empty for open-ended.

### `sweep(base, axes, metric?, max_parallel?, method?, ...)`

- `base` — a `backtest(...)` value (positional).
- `axes` — non-empty array of `axis(...)` values.
- `metric` — string; one of `total_return`, `cagr`, `sharpe`,
  `sortino`, `max_drawdown`, `win_rate`. Default `"sharpe"`.
- `max_parallel` — integer ≥ 0; `0` means
  `hardware_concurrency()` at run time. Default `0`.
- `method` — string; one of `"grid"` (default, full cartesian
  product), `"random"` (sample N configs), or `"coarse_to_fine"`
  (grid stage 1 → refine around top-K).

Axis paths for `.qe` sweeps target **top-level `let` names** —
overrides shadow the matching `let` when each cell re-evaluates
the file. See `tests/fixtures/`-adjacent `example_sweep_spy.qe`
for the canonical shape.

Method-specific kwargs:

- `method = "random"` requires `n_samples` (positive integer).
  Optional `seed` (non-negative integer) — when omitted, a stable
  hash of the axes is used so the same `.qe` file produces the
  same sample list across runs. When `n_samples` >= full grid
  size, the runner falls back to grid behavior.
- `method = "coarse_to_fine"` accepts `levels` (2..4, default 2)
  and `top_k` (positive integer, default 3). Stage 1 runs the full
  coarse grid; stage 2 refines around the top-K winners by
  interpolating midpoints between each winner and its axis
  neighbors. Integer-only axes have midpoints snapped to integers
  so `sma(close, fast)` style window kwargs keep working.
  `coarse_to_fine` only runs inside `walk_forward(... optimize =
  sweep(method = "coarse_to_fine"))` — the F4 BCKT manual
  heatmap UI doesn't fit the two-stage auto-refine model and
  rejects the method.

#### Compatibility with `signalize_universe(...)` and per-leg data

Bases whose strategy is `signalize_universe(...)` (or a manual
`portfolio(...)` whose legs each carry their own `data = ...`
template) work as sweep bases. The cell loader walks both
`base.data[]` and every leg's per-leg data template, then aligns
across the union. `base.data = []` is legal when every leg
supplies its own data.

```qe
let mom = factor("mom", expr = lag_return(close, 252))
let base = backtest(
  data = [],
  strategy = signalize_universe(mom,
    universe = universe(
      symbols = ["XLK", "XLF", "XLE"],
      data    = yahoo_template("1d", "2024-01-01", "2024-12-31")),
    top_k = 2, bottom_k = 0),
  output = output(results = "out/rotation.json"),
)

sweep(base, axes = [
  axis("execution.commission_bps", values = [1.0, 2.0, 5.0])])
```

### Channel / volatility indicators

| Builtin | Args | Description |
|---|---|---|
| `rolling_max(<expr>, n)` | expression + window | Max of `<expr>` over the trailing `n` bars. NaN during the first `n-1` calls. Backs Donchian-style breakout strategies — but note that `rolling_max(high, n)` includes the current bar, so the textbook breakout test is `close > lag(rolling_max(high, n), 1)`. |
| `rolling_min(<expr>, n)` | expression + window | Min of `<expr>` over the trailing `n` bars. Same self-inclusion caveat as `rolling_max`. |
| `atr(n)` | window only | Wilder's Average True Range over the trailing `n` bars. Reads `high` / `low` / `close` from the bar context directly — no expression arg. The first `n` bars warm up the seed average; subsequent bars apply Wilder smoothing. Backs ATR-stop strategies (`close - 3 * atr(14)` as a trailing stop). |
| `lag(<expr>, n)` | expression + window | Returns the value of `<expr>` from `n` bars ago. `n` must be a positive integer literal. NaN until the inner expression has emitted `n` valid samples (NaN bars from the inner's own warmup don't count against the lag delay). Backs textbook Donchian breakouts (`cross_above(close, lag(rolling_max(high, 20), 1))`) and price-momentum factors (`close / lag(close, 252) - 1.0`). Composes: `lag(lag(x, 2), 3)` is bar-aligned with `lag(x, 5)` once both are warm. |

```qe
signal(
  symbol = "SPY",
  # Textbook Donchian 20 / 10 — `lag(..., 1)` excludes the current
  # bar from the rolling window so the breakout test is meaningful.
  entry  = cross_above(close, lag(rolling_max(high, 20), 1)),
  exit   = cross_below(close, lag(rolling_min(low,  10), 1)),
)
```

### `signalize(factor, symbol, top_k?, bottom_k?)`

Tradeable factor sugar. Wraps a `factor(...)` value's `expr`
expression in `is_top` / `is_bottom` and returns a runnable
`SignalValue` so the same factor can be screened in
`research(...)` and traded in `backtest(...)` without
re-authoring the rank logic.

```qe
let momentum = factor("mom_close", expr = close)

research(... factors = [momentum], ...)

backtest(
  data     = [...],
  strategy = signalize(momentum, symbol = "SPY", top_k = 1),
)
```

- `factor` — positional, must be a `factor(...)` value. Reused by
  reference; the original `FactorValue` stays valid for the
  research path.
- `symbol` — required string; the leg's target ticker.
- `top_k` — optional positive integer literal (default `1`); the
  generated `entry` is `is_top(factor.expr, top_k)`.
- `bottom_k` — optional positive integer literal (defaults to
  `top_k`); the generated `exit` is `is_bottom(factor.expr,
  bottom_k)`.

For an N-symbol L/S portfolio, prefer `signalize_universe(...)`
(below) over hand-writing N `signalize(...)` legs:

### `signalize_universe(factor, universe, top_k?, bottom_k?)`

Fan a single `factor(...)` value out into an equal-weight
portfolio of N legs over a universe. Each leg gets the universe's
data template with its `symbol` filled in.

```qe
let mom = factor("mom_rsi", expr = rsi(close, 14) * -1.0)

backtest(
  data     = [],
  strategy = signalize_universe(mom,
    universe = universe(
      symbols = ["XLK", "XLF", "XLE", "XLV", "XLY",
                 "XLP", "XLI", "XLB", "XLU"],
      data    = yahoo_template("1d", "2020-01-01", "2024-12-31"),
    ),
    top_k    = 2,    # long the top 2 by factor each bar
    bottom_k = 2,    # short the bottom 2
  ),
)
```

- `factor` — positional, must be a `factor(...)` value (reusable
  across `research(...)` and `signalize_universe(...)`).
- `universe` — required; a `universe(symbols, data)` value. Same
  shape consumed by `research(...)`. The data template may be
  `yahoo_template(...)` (universe expander fills `YahooData.symbol`
  per leg) or `file("path/%s.csv")` (printf-style `%s` filled per
  leg).
- `top_k` / `bottom_k` — optional non-negative integer literals
  (defaults `1` and `top_k` respectively). Setting `bottom_k = 0`
  opts into **long-only mode**: a leg exits when it falls out of
  the top_k instead of being shorted into the bottom_k.
  Required for retail brokers without short-borrow. Applies to
  `signalize(...)` and `signalize_universe(...)` symmetrically:

  ```qe
  signalize_universe(mom, universe = u,
    top_k    = 3,
    bottom_k = 0)   # long-only — exit when no longer in top 3
  ```
- `rebalance` — optional non-negative integer (default `0` = every
  bar). When `> 0`, each leg only acts on entry / exit decisions
  at bars where `bar_index % rebalance == 0`. The cross-sectional
  pre-pass and stateful indicators still step every bar so warmup
  and rank state remain consistent — only the trade decision is
  gated. Necessary to match research-path quintile-rebalance Sharpe;
  without throttling, `signalize_universe` ranks bar-by-bar and
  generates orders-of-magnitude more turnover than the research
  analytic equivalent.

Weights default to equal (`1/N` each). `csv_url(...)` data
templates aren't supported yet — surface a clear error if you
try one.

For a multi-symbol L/S portfolio without the universe sugar, wrap
multiple `signalize(...)` legs inside `portfolio(...)`:

```qe
let momentum = factor("mom", expr = close)

backtest(
  data     = [...],
  strategy = portfolio(strategies = [
    signalize(momentum, symbol = "SPY", top_k = 1),
    signalize(momentum, symbol = "QQQ", top_k = 1),
    signalize(momentum, symbol = "IWM", top_k = 1),
  ]),
)
```

Cross-sectional primitives inside `signalize(...)` work whether
the factor's `expr` is stateless or stateful — see the next
section for the full primitive set.

### Cross-sectional primitives

The signal-context layer exposes four within-bar primitives that see
the whole universe loaded in the backtest's `MultiBarSeries`:

| Primitive | Returns | Description |
|---|---|---|
| `rank(<expr>)` | integer 0..N-1 | Ascending rank of `<expr>` for the current symbol within the universe. NaN inputs sort to the end. |
| `quantile(<expr>, q)` | integer 0..q-1 | Which quantile bucket the current symbol falls into. Bucket 0 = smallest. |
| `is_top(<expr>, k)` | bool | True if the current symbol is in the top K by `<expr>`. |
| `is_bottom(<expr>, k)` | bool | True if the current symbol is in the bottom K by `<expr>`. |

```qe
signal(
  symbol = "VOO",
  entry  = is_top(close, 3),       # top-3 by close in the universe
  exit   = is_bottom(close, 3),
)
```

As of Phase 3 the inner expression may be stateful — `rank(rsi(close,
14))`, `is_top(rolling_vol(close, 20), 3)`, etc. all work. The
Evaluator lazily allocates an N-parallel pool of inner evaluators
(one per universe symbol) so stateful indicators inside the inner
maintain per-symbol state independently. Nested cross-sectional
(e.g. `rank(rank(close))`) is still rejected — the engine pre-pass
is single-level. NaN-valued symbols (warmup) get the per-kind "no
signal" sentinel: `rank` / `quantile` return NaN; `is_top` /
`is_bottom` return 0 so the `entry_val != 0.0` gate in the strategy
correctly says "no entry".

`quantile` / `is_top` / `is_bottom` require their second arg to be
a positive integer literal.

### `weight_axis(path, values)`

Linked sweep axis for portfolio weights. Builds a single axis
whose N inner paths (`path[0]`, `path[1]`, …, `path[N-1]`)
iterate **together** per cell rather than cross-multiplying, so
each cell binds a constraint-preserving weight tuple. The weights
must sum to 1.0 (±1e-9) within each tuple — validated at parse
time.

```qe
sweep(base, axes = [
  weight_axis(
    "strategy.weights",
    values = [[0.6, 0.4], [0.5, 0.5], [0.4, 0.6]],  # 3 cells
  ),
])
```

`weight_axis` composes Cartesianly with regular `axis(...)` axes
— combining a 3-cell weight axis with a 2-value `axis("fast", …)`
gives 6 cells. It does **not** participate in
`coarse_to_fine` refinement (the sum-to-1 constraint doesn't
compose with midpoint interpolation): if `method = "coarse_to_fine"`
is used, the linked axis stays unchanged across refinement stages
while the regular axes refine.

### `paired_axis(paths, values)`

General linked-axis form. Same shape as `weight_axis` but the
paths are spelled explicitly and there's no sum-to-1.0
constraint, so it covers any case where two or more parameters
must move together rather than cross-multiplying. Numeric and
string tuples are both valid; per-row arity is enforced.

```qe
# Quarterly date-window sweep — `from` and `to` are linked.
sweep(base, axes = [
  paired_axis(
    paths  = ["from", "to"],
    values = [
      ["2025-01-01", "2025-03-31"],
      ["2025-04-01", "2025-06-30"],
      ["2025-07-01", "2025-09-30"],
      ["2025-10-01", "2025-12-31"],
    ],
  ),
])

# Constrained (fast, slow) MA pair sweep — only valid combos.
sweep(base, axes = [
  paired_axis(
    paths  = ["strategy.fast", "strategy.slow"],
    values = [[5, 20], [10, 50], [20, 200]],
  ),
])
```

Like `weight_axis`, `paired_axis` composes Cartesianly with regular
`axis(...)` axes and stays unchanged across `coarse_to_fine`
refinement stages.

### `axis(path, values | range)`

- `path` — either a top-level `let` name (`"fast"`, `"sym"`) **or** a
  dotted/bracketed path that walks into the backtest's structure:
  - **Shorthand roots**: `"execution.commission_bps"`,
    `"strategy.weights[0]"`, `"data.start"` — the first segment is one
    of `data` / `strategy` / `execution` / `output` / `date_range` /
    `sectors`, resolved against the implicit backtest reached via the
    sweep / walk_forward `base` (or a direct top-level `backtest(...)`).
  - **Explicit let prefix**: `"base.execution.commission_bps"` —
    first segment is a top-level let; remaining segments walk into
    that let's value.
  - Dotted segments index named kwargs; bracketed segments
    (`weights[0]`) index array literals. Positional args are not
    addressable — rewrite them as named kwargs (`yahoo(start =
    "2020-01-01")`) for the override to find them.
  - **Defaulted execution fields**: paths under
    `execution.{capital, commission_bps, slippage_bps, fill_model,
    compound, impact_bps_per_pct_adv}` can be overridden even when
    the source `.qe` omits the kwarg or doesn't declare an
    `execution(...)` block at all. Signal-layer paths
    (`strategy.size`, `strategy.weights[i]`) still require the
    source to spell the kwarg so the override has somewhere to
    bind.
- Second arg is either `values = [a, b, c]` (array) or a range
  expression `a..b [step c]`. Ranges inclusively cover their
  endpoint when the step lands on it exactly.

Numeric and string axes are both supported. Within one axis the
values must be homogeneous — either all numbers or all strings;
mixed arrays are rejected. Ranges are numeric-only. Strings let
you sweep over a symbol set or any other config-context name:

```qe
let sym = "SPY"
sweep(
  base,
  axes = [axis("sym", values = ["SPY", "QQQ", "IWM"])],
)
```

A string-typed axis can only target config-context positions
(`yahoo(sym, ...)`, `signal(symbol = sym, ...)`, etc.). Strings
have no meaning inside the per-bar signal expression, so a
sweep over a name that appears inside `signal(entry = …)`
fails fast with a column-aware error.

### `portfolio(strategies, weights?, names?, rebalance?)`

Combine N independent signals into one portfolio with fixed capital
weights. Goes inside `backtest(strategy = portfolio(...))`.

- `strategies` — non-empty array of `signal(...)` **OR**
  `signalize_universe(...)` **OR** nested `portfolio(...)` values.
  Nested portfolios are auto-flattened into the parent's leg
  list. One "slot" in the `strategies` array may contribute
  multiple flat legs.
- `weights` — array of non-negative numbers summing to 1.0 ±1e-6.
  **One entry per slot, not per flat leg.** When a slot is a nested
  portfolio (e.g. a 9-symbol `signalize_universe(...)`), its slot
  weight is distributed across the inner legs proportional to the
  inner weights. Default: equal across slots.
- `names` — array of non-empty display names, one per slot. For
  multi-leg slots the final flat names are auto-prefixed
  `${slot_name}_${inner_symbol}` so the attribution table stays
  distinguishable. Defaults to `"strategy_0"`, `"strategy_1"`, …
- `rebalance` — string; v1 only accepts `"never"`. Monthly /
  quarterly rebalancing is planned.

#### Nested-portfolio example

```qe
let mom = factor("mom", expr = lag_return(close, 252))

backtest(
  data     = yahoo("SPY", "1d", "2020-01-01", "2024-12-31"),
  strategy = portfolio(
    strategies = [
      signal(entry = rsi(close, 14) < 30,
             exit  = rsi(close, 14) > 70,
             symbol = "SPY"),                        # slot 0 — 1 leg
      signalize_universe(mom,
        universe = universe(
          symbols = ["XLK", "XLF", "XLE", "XLV"],
          data    = yahoo_template("1d", "2020-01-01", "2024-12-31")),
        top_k = 2, bottom_k = 0),                    # slot 1 — 4 legs
    ],
    weights = [0.5, 0.5],                            # 50% to SPY mean-revert,
                                                     # 50% spread across the 4-sector rotation
  ),
)
```

Resulting flat book: 5 legs at weights `[0.5, 0.125, 0.125, 0.125,
0.125]`. The flat names are `"strategy_0", "strategy_1_XLK",
"strategy_1_XLF", "strategy_1_XLE", "strategy_1_XLV"` (or the
user-supplied `names = [...]` slot names, suffixed by symbol).

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

See [`docs/multi-strategy.md`](multi-strategy.md) for the semantics
(why N parallel runs, why no cross-strategy netting, what
attribution actually means) and the F4 BCKT `ATTRIBUTION` panel.

### `walk_forward(base, train_window, test_window, step_window, optimize?)`

Rolling in-sample / out-of-sample validation. Top-level value
(peer of `backtest(...)` / `sweep(...)`).

- `base` — a `backtest(...)` value (positional or kwarg). The
  base's `strategy` can be either a single `signal(...)` or a
  `portfolio(...)` of N children; the runner uses the combined
  per-window equity curve to compute OOS metrics.
- `train_window`, `test_window`, `step_window` — required
  duration strings. Suffixes: `ns`/`us`/`ms`/`s`/`m`/`h`/`d`/`w`/
  `mo`/`y`; calendar `mo` = 30d, `y` = 365d (windows align by bar
  index, so the slop is bounded). `step_window` is named to avoid
  the `step` keyword used by range syntax.
- `optimize` — optional `sweep(...)`. When set, each train slice
  re-runs the sweep, picks the best params by `optimize.metric`,
  and applies them to the test slice.

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
  train_window = "365d",
  test_window  = "90d",
  step_window  = "90d",
  optimize     = sweep(base,
    axes   = [axis("fast", values = [5, 10, 15, 20]),
              axis("slow", values = [50, 100, 150, 200])],
    metric = "sharpe",
  ),
)
```

Omit `optimize` for fixed-parameter rolling validation — the same
parameters run on every test slice, and the F4 BCKT view shows the
stitched OOS curve without IS-vs-OOS comparison or `PARAM DRIFT`:

```qe
walk_forward(base,
  train_window = "365d",
  test_window  = "90d",
  step_window  = "90d",
)
```

Schema v5's `best_params` field is `{}` for fixed runs — that's how
downstream consumers detect fixed vs. optimized.

Partial tail windows (where the test slice would run past the last
bar) are silently discarded so OOS metrics aren't distorted by an
under-length last segment. See [`docs/walk-forward.md`](walk-forward.md)
for IS-OOS gap reading and the F4 BCKT walk-forward view.

## Live trading builtins

### `live(base, broker, symbols?, journal_dir?, ibkr?, risk_*?)`

Top-level value consumed by `qe_daemon start` (EPIC-62). Wraps a
`backtest(...)` block — byte-identical to what you'd run offline —
with the live-only knobs needed to drive the same strategy against
a real broker.

Required:
- `base` — a `backtest(...)` value. Same strategy + execution as
  the offline run.
- `broker` — one of `"ibkr-paper"`, `"ibkr-live"`, `"alpaca-paper"`,
  `"alpaca-live"`. Only the IBKR variants drive a running daemon
  today; Alpaca parses but exits 4 with a clear error.

Optional:
- `symbols` — array of strings. Superset of `base.data` symbols
  allowed (the live aggregator may want a reference feed the
  strategy logs but doesn't trade). Defaults to derived-from-
  `base.data`.
- `journal_dir` — override the JSONL journal root. Default:
  `~/Library/Application Support/qe_daemon/state/` (macOS) or
  `$XDG_DATA_HOME/qe_daemon/state/` (Linux).
- `ibkr` — `ibkr_connection(...)` value (see below). Defaults to
  the standard local Gateway layout.
- `risk_max_position_per_symbol`, `risk_max_gross_exposure_usd`,
  `risk_max_daily_loss_usd`, `risk_max_orders_per_minute` — pre-
  trade risk knobs. Each defaults off; opt in deliberately.
  `risk_max_daily_loss_usd` is a NEGATIVE number (e.g. `-1500.0`);
  hitting it trips the kill-switch for the rest of the session.
  See [`live-trading-runbook.md`](live-trading-runbook.md) for
  the picking-limits heuristic.

```qe
live(
  base   = base,
  broker = "ibkr-paper",
  symbols = ["SPY"],
  ibkr = ibkr_connection(host = "127.0.0.1", port = 7497, client_id = 1),
  risk_max_position_per_symbol = 200,
  risk_max_gross_exposure_usd  = 60_000,
  risk_max_daily_loss_usd      = -1500,
  risk_max_orders_per_minute   = 10,
)
```

### `ibkr_connection(host?, port?, client_id?, account?)`

Carries the IBKR Gateway / TWS Desktop connection parameters for
the daemon. All four args are optional; the defaults match the
public IBKR Gateway documentation:

| Arg         | Type   | Default                                |
|-------------|--------|----------------------------------------|
| `host`      | string | `"127.0.0.1"`                          |
| `port`      | int    | `7497` for `ibkr-paper`, `7496` for `ibkr-live` |
| `client_id` | int    | `1`                                    |
| `account`   | string | `""` (daemon picks first available)    |

Port reference:
| Endpoint               | Port |
|------------------------|------|
| Paper IB Gateway       | 4002 |
| Paper TWS Desktop      | 7497 |
| Live IB Gateway        | 4001 |
| Live TWS Desktop       | 7496 |

Standalone use (not as the `ibkr` kwarg of `live(...)`) is rejected
at parse time — it produces an `IbkrConnSpec` that no other builtin
consumes.

## Factor research builtins

A second top-level value, `research(...)`, drives the **factor**
research path. It evaluates exactly like the other top-level values
but is consumed by `qe_factor` (not `qe_run`) and emits
`factor_report.json` instead of `results.json`. The F7 FCTR
dashboard panel mtime-watches that file. See
[`factor-research.md`](factor-research.md) for the workflow and the
report schema; this section is the grammar reference.

### `research(universe, factors, horizons?, quantiles?, rebalance?, walk_forward?, output?)`

Cross-sectional ranking research over a universe of symbols.

- `universe` — one `universe(...)` value (positional or kwarg).
- `factors` — non-empty array of `factor(...)` values.
- `horizons` — array of positive integers, forward-return windows in
  bars. Default `[1, 5, 20]`.
- `quantiles` — integer bucket count for the long-short. Default `5`.
  Must satisfy `2 * quantiles ≤ |symbols|`.
- `rebalance` — long-short holding period in bars. Default `5`.
- `walk_forward` — optional `walk_forward_ic(...)` (see below).
- `output` — `output(report = "...")` writes
  `factor_report.json` to disk; omit it and `qe_factor` prints the
  JSON to stdout.

Minimal example:

```qe
research(
  universe = universe(
    symbols = ["XLK", "XLF", "XLE", "XLV", "XLY",
               "XLP", "XLI", "XLB", "XLU"],
    data    = yahoo_template("1d", "2015-01-01", "2025-01-01"),
  ),
  factors = [
    factor("momentum_20", expr = (close / sma(close, 20)) - 1.0),
  ],
  output = output(report = "out/momentum_research.json"),
)
```

See [`factor-research.md` § Quick start](factor-research.md#quick-start)
for the long-form walk-through and IC interpretation.

### `factor(name, expr)`

A named factor expression. `name` is a positional string used as the
report key + the F7 dropdown label; `expr` is a **signal-context
expression** (same grammar / variables as `signal(entry = ...)`)
that must reduce to a *scalar* per bar — booleans collapse to 0/1
and throw away rank, defeating the point.

```qe
factor("rsi_14_inv", expr = rsi(close, 14) * -1.0)
factor("close_minus_sma10", expr = close - sma(close, 10))
```

See [`factor-research.md` § Authoring rules](factor-research.md#authoring-rules)
for the scalar-vs-boolean trap, NaN propagation, and let-binding
tricks across factors.

### `universe(symbols, data)`

Defines the cross-section qe_factor loads.

- `symbols` — non-empty array of strings (e.g. SPDR sector tickers).
- `data` — either `yahoo_template(...)` (per-symbol Yahoo fetch) or
  `file(path)` where `path` contains `%s` as the symbol placeholder
  (e.g. `"data/csv/factor_%s.csv"`).

Symbols are loaded sequentially and aligned to the intersection of
their trading days; mismatched coverage is silently truncated.

```qe
universe(
  symbols = ["XLK", "XLF", "XLE"],
  data    = yahoo_template("1d", "2015-01-01", "2025-01-01"),
)
```

### `yahoo_template(resolution, start, end?, cache_dir?)`

A *template* DataSpec — same shape as `yahoo(...)` but with the
symbol *deferred*. `qe_factor` substitutes each universe symbol in
turn. Only valid inside `universe(data = ...)`.

- `resolution` — `"1d"`, `"1h"`, `"1m"`, `"1wk"`, `"1mo"`.
- `start` — ISO `YYYY-MM-DD`.
- `end` — optional ISO date; defaults to "today" at load time.
- `cache_dir` — optional local cache directory override.

### `walk_forward_ic(window_bars, step_bars)`

Opt into rolling per-window IC analysis. Goes inside
`research(walk_forward = ...)`. Both args are positional or kwarg.

- `window_bars` — positive integer; rolling window length. On daily
  bars, `252` ≈ one year.
- `step_bars` — positive integer; window stride. `21` ≈ monthly.

For each `(factor, horizon)`, qe_factor runs `ic_analysis` once per
window and emits a `walk_forward` block on every `ic[]` entry of
the v3 `factor_report.json`. The F7 FCTR panel renders it as a
rolling-IC line below the per-bar XS IC plot.

```qe
research(
  universe = universe(
    symbols = ["XLK", "XLF", "XLE", "XLV", "XLY"],
    data    = yahoo_template("1d", "2015-01-01", "2025-01-01"),
  ),
  factors      = [factor("rsi_14_inv", expr = rsi(close, 14) * -1.0)],
  horizons     = [5, 20, 60],
  walk_forward = walk_forward_ic(window_bars = 252, step_bars = 21),
  output       = output(report = "out/wf_report.json"),
)
```

See [`factor-research.md` § Walk-forward IC](factor-research.md#walk-forward-ic)
for how to read regime breaks, signal decay, and the trend-slope
badge.

## Signal-layer functions

Authoritative reference: [`dsl-grammar.md`](dsl-grammar.md). The
list below is the subset most commonly used inside `signal(...)`
and `factor(...)` — see `dsl-grammar.md` for per-call semantics,
NaN propagation, and the `min_history` inference rules.

DSL functions, positional args only:

- **Stateful**: `sma(x, window)`, `ema(x, window)`, `rsi(x, period)`,
  `cross_above(a, b)`, `cross_below(a, b)`, `lag(x, n)`,
  `rolling_max(x, n)`, `rolling_min(x, n)`, `atr(n)`.
  `window`/`period`/`n` must be a positive integer literal
  (constant-folded — let-bound integers work because they're inlined
  at signal-extraction time).
- **Pure**: `abs(x)`, `max(a, b)`, `min(a, b)`.
- **Forecasting** (`predict_return`, `lag_return`,
  `rolling_vol`, `rolling_zscore`) — documented in
  [`forecasting.md`](forecasting.md).

## Walkthrough — writing your first `.qe`

1. **Bootstrap a workspace**: launch `qe_dashboard`. On a fresh
   install it drops five `example_*.qe` starters into
   `<workspace>/backtests/`, one `sweeps/example_sweep_spy.qe`, and
   a `positions/example_positions.json`.
2. **Open one in F3 WKSP** and Cmd+S — that forks `qe_run` on the
   file, streams stdout into the bottom log pane, and F4 BCKT
   auto-refreshes off the resulting `out/<name>.json`.
3. **Edit a constant** — change `let fast = 10` to `let fast = 20`,
   Cmd+S, watch the equity curve and trade count change.
4. **Compose a new signal**:
   ```qe
   let rsi_overbought = rsi(close, 14) > 70
   let above_ma       = close > sma(close, 50)

   backtest(
     data     = yahoo("SPY", "1d", "2024-01-01", "2025-01-01"),
     strategy = signal(
       entry  = above_ma and rsi(close, 14) < 30,
       exit   = rsi_overbought,
       symbol = "SPY",
     ),
     execution = execution(capital = 50_000),
     output    = output(results = "out/spy_rsi.json"),
   )
   ```
   Save → it runs. The let bindings keep the entry/exit expressions
   readable; `rsi(close, 14)` evaluating twice is harmless because
   the per-bar evaluator updates each call site's RSI instance every
   bar regardless.

## Errors

Every parse / type / lookup error carries a 1-based column number
and (for file loads) the file path. The dashboard surfaces the same
string in its banner + F3 WKSP log pane. Three representative
shapes — every example below is exercised in `tests/test_dsl_config_eval.cpp`
or `tests/test_qe_loader.cpp` so the format can't silently drift:

**Missing required kwarg** — typo or forgotten `entry =`:

```
file: backtests/example_ma_spy.qe: expression: column 14:
  signal: missing required keyword argument 'entry'
```

**Wrong variable name in a signal expression** — typo on `close`:

```
file: backtests/example_ma_spy.qe: expression: column 23:
  inside signal expression: unknown variable 'klose'
```

**Unknown top-level function** — typo on a builtin:

```
file: example.qe: expression: column 1:
  unknown function 'frobnicate'
```

Other common shapes you'll see in practice: `unknown name '<x>'`
(unbound let lookup), `yahoo: missing required argument 'start'`
(missing positional), `unknown metric '<x>'` (sweep `metric =`
not in the allowed list), `range step cannot be zero`, and `at
least one axis` (empty `sweep(axes = [])`). Format is always
`file: <path>: <node>: column <N>: <msg>`.

## What stays JSON

- `results.json` — engine output, machine-generated.
- `~/Library/Application Support/qe_dashboard/config.json` —
  auto-managed dashboard prefs.
- `positions/*.json` — broker / portfolio snapshots; these are
  *input data*, not user-authored config.

## Reference

- Lexer / parser / AST: `include/qe/dsl/{lexer,parser,ast}.hpp` +
  `src/dsl/{lexer,parser}.cpp`
- Config evaluator: `include/qe/dsl/config_eval.hpp` +
  `src/dsl/config_eval.cpp`
- Value types: `include/qe/dsl/config_value.hpp`
- File loader: `include/qe/io/qe_loader.hpp`
- Strategy entry: `qe::strategy::make_expression_strategy` in
  `include/qe/strategy/factory.hpp`
- Tests: `tests/test_dsl_lang.cpp`, `tests/test_dsl_config_eval.cpp`,
  `tests/test_qe_loader.cpp`, `tests/test_qe_run_smoke.cpp`
  (the `[qe]` test case), `tests/test_qe_sweep_runner.cpp`.
- Sweep runner: `include/qe/sweep/qe_runner.hpp` +
  `src/sweep/qe_runner.cpp`.
