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
    // Target ~2 % of equity at risk per bar, in SHARES:
    //   (0.02 * equity) / (per-bar return vol * price)
    // realized_vol is a fraction; multiplying by close puts the
    // denominator in dollars per share, which is what `size` needs.
    size  = (0.02 * 100000.0) / (realized_vol(close, 20) * close),
  )
  ```

  This example used to read `size = 0.02 / rolling_vol(close, 20)` and
  it was wrong twice over: `rolling_vol(close, n)` is the std-dev of
  the **dollar price level**, not of returns (so it sized inversely to
  price rather than to risk), and dividing a bare `0.02` by a dollar
  quantity does not produce a share count. See
  [Volatility: `rolling_vol` vs `realized_vol`](#volatility-rolling_vol-vs-realized_vol).

  Expression results that are `NaN` (warmup), `0`, or negative
  **skip the trade** — same convention as `entry` / `exit`
  evaluating false.

  **Live parity (EPIC-84 T84.1).** The daemon evaluates the same
  expression at entry time against the same bar context, so a
  stateful indicator inside it steps once per bar exactly as it does
  in the backtest, and the same NaN / non-positive skip applies. The
  result is still capped by the leg's capital budget and floored to
  the venue's lot granularity (`execution(whole_shares = ...)`) —
  an expression is a *request*, not an override of the budget.

  Before v0.3.1 the live engine had no branch for this arm at all: a
  leg kept the `1.0` default and traded one share, with nothing in
  the log to say so. Check any config with
  `qe_daemon validate <config>.qe`, which prints each leg's sizing
  mode without contacting a broker. Each entry is also defensively capped at 100 %
  of current equity at the most recent mark price; orders that
  would exceed it are scaled down. Compounding (`compound = true`)
  is suppressed when `size` is an expression — the user owns the
  equity-scaling math inside the expression.

### Volatility: `rolling_vol` vs `realized_vol`

These two do different things and the difference has cost real money.

| Call | What it returns | Units |
|---|---|---|
| `rolling_vol(x, n)` | sample std-dev of the last `n` values of **`x` itself** | whatever `x` is in |
| `realized_vol(x, n)` | sample std-dev of the last `n` **1-bar simple returns** of `x` | dimensionless fraction |

So `rolling_vol(close, 20)` is the standard deviation of the price in
**dollars**. It is not a volatility in the sense anyone means when they
say volatility. Measured across nine universes:

```
rankcorr(rolling_vol(close, 20), close)                = +0.815
rankcorr(rolling_vol(close, 20), true return vol)      = +0.118
```

which makes `rolling_vol(close, 20) * -1.0` — a shape that looked like
a low-volatility factor and appeared throughout a research corpus —
approximately `-close`, i.e. **"buy the cheapest names"**. An audit
reproduced that factor's headline t-stat to within 2 % from a
price-level null control alone. It was never two measurements.

Use `realized_vol(x, n)` when you want return volatility:

```qe
realized_vol(close, 20)                       // per-bar return vol
realized_vol(close, 20) * 15.8745             // annualized, 252 bars/yr
```

`realized_vol` is deliberately **not** annualized — it does not know
your bar cadence, and guessing it is exactly the mistake that inflated
every long-short Sharpe in the same corpus. Multiply by
`sqrt(bars_per_year)` yourself.

`realized_vol(x, n)` is bit-identical to
`rolling_vol(lag_return(x, 1), n)`, which already worked; the builtin
exists so the correct form is discoverable rather than folklore.
Warmup is `n + 1` bars (a 1-bar return needs 2 bars, then `n` of them).

**`rolling_vol`'s behaviour is unchanged and will stay unchanged** — a
live deployment trades it, and silently redefining a function under a
running strategy is its own failure class. Instead, applying
`rolling_vol` directly to a bare price variable (`close`, `open`,
`high`, `low`) emits a non-fatal **config lint** at load:

```
[warning] config lint: my_factor.qe: line 12 col 22: rolling_vol(close, n)
measures the std-dev of the PRICE LEVEL in dollars, not of returns. ...
For return volatility write realized_vol(close, n) — bit-identical to
rolling_vol(lag_return(close, 1), n). If the dollar dispersion IS what
you meant, write rolling_vol(price_level(close), n) to say so and
silence this lint.
```

The config still loads and still runs. `qe_run --check` surfaces the
lint and exits `0`. If you genuinely want dollar dispersion, wrap the
argument in `price_level(x)` — an exact identity that costs nothing at
runtime and marks the intent:

```qe
rolling_vol(price_level(close), 20)   // deliberate, no lint
```

#### Going live is where the lint becomes fatal

A warning was not enough. A paper deployment carried this exact lint in
its startup log every session and traded on it for six weeks; a
diagnostic nobody acts on is indistinguishable from no diagnostic. So
since 0.3.4 the lint is a **hard refusal at the live boundary only**:

| Tool | Behaviour |
|---|---|
| `qe_run`, `qe_factor`, sweeps, walk-forward | warn and run — unchanged |
| `qe_daemon validate` | **exit 4**, config not deployable |
| `qe_daemon start` | **exit 4**, before any socket is opened |

`qe_run` is deliberately left alone: reproducing what a legacy config
actually did is a thing you have to be able to do, and refusing to
backtest it would make the record unauditable.

Two ways past the refusal, and the escape hatch above is the good one —
`rolling_vol(price_level(close), 20)` is the same arithmetic to the
last bit, so a deployed strategy can adopt it with no change to its
results. `--allow-price-level-vol` is the other: it starts the daemon
and prints the whole refusal at ERROR on every startup, so the log
never stops saying what is running.

`rolling_zscore(close, n)` is deliberately **not** linted:
`(close - mean) / std` is dollars over dollars, hence dimensionless,
and a perfectly reasonable standardized mean-reversion factor. It does
not carry this defect.

### `yahoo(symbol, resolution, start, end?, cache_dir?)`

Fetches from the Yahoo Finance v8 chart endpoint. `symbol` and `start`
are required; `resolution` defaults to `"1d"` (also accepts `"1h"`,
`"1m"`, `"1wk"`, `"1mo"`). All five can be positional or keyword.

**Prices come back on a total-return basis.** Yahoo ships an
`adjclose` column alongside the raw quote; the loader now uses it.
Per bar, `ratio = adjclose / close`, then `close` is assigned
`adjclose` and `open` / `high` / `low` are multiplied by the same
`ratio`. All four columns therefore stay on one consistent basis,
which matters because `fill_model = "next_open"` fills at an open that
must be comparable to the close the signal was computed on.

Three consequences worth knowing before you compare against old
results:

- **The ratio is the dividend component only.** Yahoo pre-applies
  split adjustment to the quote columns before sending them, so a 2:1
  split shows up as an ordinary day (MCHP's 2021-10-13 moved −0.35 %,
  not −50 %). Nothing here removes a split jump, because there was
  never a split jump.
- **`volume` is left exactly as sent** — a split-adjusted share count.
  It is not rescaled by `1/ratio`. A dividend does not change share
  count, so doing that would fabricate an ADV trend of up to +43 % at
  the oldest bar of a high-yield name, injected into a variable every
  factor expression can read. The trade-off is that `pct_adv` in the
  market-impact model slightly **over**states friction on a
  dividend payer early in the window — the safe direction.
- **A non-payer is bit-identical.** When `adjclose == close` on every
  bar the loader writes nothing at all. Dividend payers move; their
  11-year totals move a lot (HOMB: 97.7 % → 162.3 %).

The same adjustment runs on `file(...)` and `csv_url(...)` when
`format = "yahoo"`, so a CSV and a fetch of the same symbol agree.
`format = "generic"` is never adjusted. If Yahoo's `adjclose` is
missing on a settlement interval (`1d` / `1wk` / `1mo`), or its length
disagrees with the quote arrays, or a ratio lands outside `(0, 1.5]`,
the load **fails** naming the symbol rather than returning a mix of
adjusted and raw bars. Intraday intervals have no `adjclose` and are
correctly left alone.

`end` is optional to the evaluator but **required in practice** —
`qe_run` rejects an empty `end` with `data.symbol / data.start /
data.end are required when source=yahoo`. Nothing substitutes
today's date. Same rule as `yahoo_template(...)`.

### `file(path, format?)`

Local CSV. `format` defaults to `"yahoo"` (the columnar layout
yfinance writes) — which means its `Adj Close` column **is** applied,
exactly as on the `yahoo(...)` fetch path above. A yfinance CSV and a
`yahoo(...)` fetch of the same symbol therefore agree. Pass
`format = "generic"` for a file that is already on whatever basis you
want, or whose `Adj Close` you do not trust.

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

### `execution(capital?, commission_bps?, slippage_bps?, impact_bps_per_pct_adv?, fill_model?, record_forecast?, compound?, whole_shares?, entry_schedule?, exit_schedule?)`

All keyword, all optional. Defaults: `capital = 100_000`, others
zero, `fill_model = "next_open"` (also accepts `"next_close"`),
`compound = false`, `impact_bps_per_pct_adv = 0`,
`record_forecast = false`, `whole_shares = false`, no schedules.

Four of the ten are documented in full elsewhere:

| Kwarg | Where |
|---|---|
| `record_forecast` | [`forecasting.md`](forecasting.md) — emits the per-bar forecast table into `results.json`. |
| `entry_schedule` / `exit_schedule` | [`staged-entry.md`](staged-entry.md) — `staged_entry(...)` / `staged_exit(...)` slice a live order across a window. **Live-only**; the backtest fills the whole order at the next bar. |
| `whole_shares` | below. **Live-only.** |

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

**`whole_shares = true`** floors every entry quantity to an integer
share count. Opt in for venues or accounts that reject fractional
orders — IBKR's paper account answers TWS `10243` to every
fractional-sized API order even though it accepts whole-share orders
from the same account. Default `false` keeps the 0.0001-share
granularity IBKR documents for fractional-enabled accounts.

> **`whole_shares` is LIVE-ONLY, and that divergence is deliberate.**
> Only `qe::live::LiveEngine` reads it (via `capped_entry_qty`); the
> backtest engine ignores it completely and keeps sizing in
> fractional shares. So a `whole_shares = true` config is *not* a
> bar-for-bar replica of its own backtest: the live book rounds each
> leg down, the backtest doesn't. The gap is bounded by one share per
> leg per entry, which is negligible on a $250 leg of a $30 stock and
> is the whole position on a $250 leg of a $400 stock. The divergence
> was accepted rather than mirrored into the backtest because it is a
> *venue* constraint, not a strategy property — the same `.qe` may be
> gated offline and then deployed to two venues with different
> fractional support. Size legs so the rounding is noise, or gate the
> strategy at the capital level you will actually deploy.
>
> Concretely, at `whole_shares = true` a leg whose budget can't
> afford one share submits **nothing** and logs a skip naming the
> real minimum lot (`1 share`, not `0.0001 shares`). That is the
> mechanism behind the $25k paper deploy that never filled.

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

### `signalize_universe(factor, universe, top_k?, bottom_k?, rebalance?, size?, weighting?)`

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
- `exit_rank` — optional positive integer (default `top_k`).
  **Rotation hysteresis**: entry stays `top_k`, but a name is held
  until it falls past `exit_rank`. Requires `bottom_k = 0`.

  Without it a name sitting on the boundary is sold and re-bought as
  its rank oscillates across a single position — two spreads and two
  commissions for no change in view. At `rebalance = 1` on a 30-name
  universe that is usually the dominant cost.

  ```qe
  signalize_universe(score, universe = u,
    top_k     = 10,
    bottom_k  = 0,
    exit_rank = 12)   # enter in the top 10, hold until out of the top 12
  ```

  Refused if `exit_rank < top_k` (a narrower exit band than the entry
  band sells a name the same bar it was bought), if it exceeds the
  universe size (`is_top(f, k)` with `k > n` is false for *every*
  symbol, so the negated exit would fire every bar and the strategy
  would flatten itself continuously), or if `bottom_k > 0` (there the
  exit is `is_bottom(factor, bottom_k)` and the gap between the two
  bands *is* the hysteresis — a second one would be ambiguous).

  **It costs gross exposure, under every weighting.** Up to
  `exit_rank` legs can be open against a budget sized for `top_k`:

  - `weighting = "equal_k"` — gross reaches `exit_rank / top_k`,
    1.2× at 10/12.
  - `weighting = "inverse_vol"` — **`target_gross` does not bound the
    book.** The allocator scales the *current* `top_k` selection, and
    nothing resizes a position once it is open, so the names that have
    slid to rank `top_k+1 … exit_rank` keep their entry notional
    *outside* that sum. Gross runs at about
    `target_gross × exit_rank / top_k` with even weights and at most
    `exit_rank × max_weight` with every open leg at the cap. See
    ["`inverse_vol` does not normalize the held
    book"](#inverse_vol-does-not-normalize-the-held-book) below.
  - `weighting = "equal_universe"` — `1/n` per leg, so `exit_rank ≤ n`
    legs can never exceed 1×. The only combination hysteresis cannot
    lever up.

  The loader warns with the exact multiples in the first two cases.
- `rebalance` — optional non-negative integer (default `0` = every
  bar). When `> 0`, each leg only acts on entry / exit decisions
  at bars where `bar_index % rebalance == 0`. The cross-sectional
  pre-pass and stateful indicators still step every bar so warmup
  and rank state remain consistent — only the trade decision is
  gated. Necessary to match research-path quintile-rebalance Sharpe;
  without throttling, `signalize_universe` ranks bar-by-bar and
  generates orders-of-magnitude more turnover than the research
  analytic equivalent.
- `size` — optional positive number. Overrides the default
  budget sizing (below) with a literal per-entry share count.
  `size = 1` reproduces the pre-EPIC-83 behaviour exactly. Numeric
  only: `signalize_universe` args evaluate in config context, so a
  per-bar expression has no home here — author those through
  `signal(size = ...)`.
- `weighting` — optional string: `"equal_k"` (default),
  `"equal_universe"`, or `"inverse_vol"`. The first two select the
  denominator of each leg's static **sizing budget** (below).
  Anything else is a load-time error.
- `weight_expr` / `max_weight` / `target_gross` — the `"inverse_vol"`
  parameters. **Every name gets the same risk rather than the same
  capital**: weights proportional to `1 / weight_expr` across the
  *selected* names, each capped at `max_weight`, scaled to
  `target_gross`, recomputed every bar.

  ```qe
  let vol = factor("vol", expr = realized_vol(close, 60))

  signalize_universe(score, universe = u,
    top_k        = 10,
    bottom_k     = 0,
    exit_rank    = 12,
    weighting    = "inverse_vol",
    weight_expr  = vol,
    max_weight   = 0.15,   # no name above 15 % of capital at entry
    target_gross = 1.0)    # the top 10 sum to 1.0x — NOT the whole book:
                           # exit_rank = 12 holds two more, ~1.2x typical,
                           # 1.8x worst case. See the subsection below.
  ```

  `weight_expr` is required under `"inverse_vol"` and refused under
  any other weighting — a risk model that is silently not being used
  is worse than none. `size` and `"inverse_vol"` both decide the
  quantity, so specifying both is an error. `max_weight` is a fraction
  of capital in `(0, 1]`, not a share count.

  Capping frees a remainder, which is redistributed among the names
  still under the cap — and that can push a second name over, so it
  iterates to a fixed point. When `max_weight * top_k < target_gross`
  the target is unreachable (10 names at a 5 % cap reach 50 %); every
  name sits at the cap, the book runs under-deployed, and the loader
  warns with the exact shortfall rather than levering up to hide it.

  A name whose `weight_expr` is NaN, zero, negative or infinite gets
  **no allocation at all** — a missing risk estimate is not a small
  risk. A tiny positive one is floored rather than dropped. If no
  selected name has a usable estimate, nothing is bought.

  Both the score factor and `weight_expr` may contain the
  cross-sectional normalizers, and the allocator reads the value you
  wrote — a `xs_winsorize`d risk estimate weights on the winsorized
  numbers, and a composite score selects on the composite. Mind the
  sign if you normalize the risk: `xs_zscore` and `xs_winsorize` return
  a *z-score*, which is negative for every below-average name, and a
  non-positive risk estimate gets no allocation per the rule above.
  Shift it into positive territory (`2.0 + xs_winsorize(vol, 1.0)`) or
  normalize with `xs_rank_pct` and keep clear of the zero at the bottom
  of its range.

  Backtest and live compute this independently and identically: the
  cross-sectional pre-pass hands *every* leg the whole universe's
  scores and risk values, so each leg runs the same allocation over
  the same inputs. There is no shared intermediate for the two to
  disagree about.

  `target_gross` bounds the *selection*, not the book — see the
  subsection below before pairing it with `exit_rank`.

`csv_url(...)` data templates aren't supported yet — surface a
clear error if you try one.

#### Leg sizing — budget-sized by default (EPIC-83)

> **Breaking behaviour change in EPIC-83. Read this before
> redeploying any `signalize_universe` strategy — it changes the
> backtest result at every capital level.**

Each leg is **budget-sized**: at entry it buys as many shares as its
capital budget affords at that bar's mark, rather than a fixed share
count. Before EPIC-83 every leg carried `size = 1.0`, i.e. **one
share**, so the book was capped at N shares regardless of capital and
the legs were weighted by *share price* instead of by conviction. On
the $25k paper deploy that left ~7 % of capital deployed.

Two independent weight vectors come out of the builtin, and mixing
them up is the easiest way to misread a result:

| Vector | Value | Sums to | Used for |
|---|---|---|---|
| `weights` (accounting) | `1/n` | exactly `1.0` | splitting capital across legs for attribution, `combined_equity`, `initial_capital`, every % metric |
| `budget_weights` (sizing) | `1/k` under `equal_k`, `1/n` under `equal_universe` | `n/k ≥ 1` — **not** a distribution, never renormalized | how many shares each entry buys |

`k = top_k + bottom_k`, the number of slots the strategy is designed
to hold at once. `k == 0` is a load-time error — it is a denominator.
The accounting split is untouched by `weighting`, so
`combined_equity[0]` still equals `execution.capital` in both modes.

Per-leg entry budget, for a 30-symbol universe with
`top_k = 10, bottom_k = 0`:

| Mode | budget @ `capital = 1_000` | budget @ `capital = 25_000` | gross with all 10 slots filled |
|---|---|---|---|
| pre-EPIC-83 (`size = 1`) | 1 × mark | 1 × mark | `10 × mark` — independent of capital |
| `weighting = "equal_universe"` | $33.33 | $833.33 | `k/n` ≈ 33 % of capital |
| `weighting = "equal_k"` (default) | **$100.00** | **$2,500.00** | ≈ 100 % of capital |

So the default triples per-leg notional relative to the legacy
`1/n` split, on top of replacing "one share" with "a budget".

**Any strategy gated before EPIC-83 must be re-gated.** The old
deployment evidence — Sharpe, drawdown, turnover, capacity, every
gate that was run against a `signalize_universe` config — was
produced under one-share legs and does not transfer. Re-run the
full gate sequence at the capital level you intend to deploy.

Escape hatches, in increasing order of bluntness:

```qe
signalize_universe(mom, universe = u, top_k = 10, bottom_k = 0)
#   → budget-sized, 1/k budget. The EPIC-83 default.

signalize_universe(mom, universe = u, top_k = 10, bottom_k = 0,
  weighting = "equal_universe")
#   → budget-sized, but the legacy 1/n budget.

signalize_universe(mom, universe = u, top_k = 10, bottom_k = 0,
  size = 1)
#   → literally the pre-EPIC-83 config: one share per leg.
```

##### `equal_k` with `bottom_k > 0` can exceed 1x gross

`equal_k` budgets each leg as if at most `k` legs are open at once.
That is **exact for a long-only rotation** (`bottom_k = 0`): a leg
exits the moment it leaves the top `k`, so the open count never
exceeds `k`.

With `bottom_k > 0` it is an under-estimate. A long leg holds until
it slides all the way into the *bottom* `bottom_k`, so it can sit
outside the top `top_k` and still be open. Up to `n - bottom_k` legs
can hold simultaneously, and gross exposure reaches
`(n - bottom_k) / k` × capital. For the 9-symbol
`top_k = 2, bottom_k = 2` example above that is `7/4 = 1.75x` —
i.e. the book borrows 75 % on margin.

Config-eval **warns** in that case rather than rejecting; the shape
is legitimate (a hysteresis band is a real design), it just has to
be a deliberate choice. The warning names the actual multiple:

```
signalize_universe: weighting="equal_k" budgets each leg at 1/4 (25.0%)
of capital, but bottom_k=2 keeps a leg open until it falls into the
bottom 2 — up to 7 of the 9 legs can hold at once, so gross exposure
can reach 1.75x capital. Pass weighting = "equal_universe" for the 1/n
split that can never exceed 1x, or bottom_k = 0 for the long-only
rotation this budget assumes.
```

Three ways to answer it: accept the leverage, set
`weighting = "equal_universe"` (`1/n` can never exceed 1x), or set
`bottom_k = 0` for the long-only rotation the `1/k` budget assumes.
Nothing is enforced in code — a margin call is a broker-side event
the engine cannot see.

##### `inverse_vol` does not normalize the held book

`target_gross` is often read as a leverage cap. **It is not.** It is
the sum of the weights the allocator hands to the names that are in
the top `top_k` *on this bar*. With `exit_rank = top_k` those are the
same set and the reading holds. With `exit_rank > top_k` it does not.

Neither runtime resizes a position that is already open — the backtest
enters under `if (!in_position_)` and live under
`!in_position && !pending`, and there is no other write path to an
existing holding. So a name that slides to rank `top_k+1 … exit_rank`
is still held, still carries the notional it entered with, and is not
a term in the sum the allocator scales. Meanwhile a full
`target_gross` is allocated across the fresh top `top_k`.

For `top_k = 10, exit_rank = 12, max_weight = 0.15, target_gross = 1.0`:

| | gross |
|---|---|
| even weights (no vol dispersion) | `target_gross × exit_rank / top_k` = **1.2x** |
| ceiling | `target_gross + (exit_rank − top_k) × max_weight` = **1.3x** |

The ceiling is *not* `exit_rank × max_weight`: that would need every
open leg at the cap simultaneously, and the current selection cannot be
— `weights_for` scales it to exactly `target_gross`. Only the overhang
is unbounded by that sum, and each overhang name entered at no more
than `max_weight`.

The loader warns with both bounds:

```
signalize_universe: weighting="inverse_vol" scales only the CURRENT
top_k=10 selection to target_gross=1.00x, but exit_rank=12 holds a name
until it falls past rank 12. Nothing resizes a position once it is open,
so the up to 2 names that have slid out of the top 10 keep their entry
notional and sit OUTSIDE that sum — gross exposure runs at about 1.20x
with even weights and at most 1.30x (the overhang enters at no more than
max_weight=0.150 apiece). target_gross does NOT bound the book under
hysteresis.
Set exit_rank = top_k to turn hysteresis off, or lower target_gross /
max_weight to pay for the overhang.
```

Three ways to answer it, same as above: accept the leverage knowingly,
set `exit_rank = top_k` (no hysteresis, no overhang, more turnover), or
size `target_gross` and `max_weight` so that the *worst case* — not the
selection — is the exposure you intend. A fourth would be to normalize
over the union of held and selected names; that needs a resize path
neither runtime has, and it is recorded as a known limitation in
`EPICS/EPIC-89-defensive-momentum.md` rather than half-built.

One consequence worth naming: the **backtest has no backstop either**.
`MultiPortfolio` lets cash go negative, and the T60.2 equity clamp is
deliberately skipped for budget-sized entries. An equity curve produced
under this combination is itself levered — it is not evidence that the
leverage is affordable.

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

The signal-context layer exposes seven within-bar primitives that see
the whole universe loaded in the backtest's `MultiBarSeries`. Four
**select**; three **normalize** (see below).

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

#### Cross-sectional normalization

The four above answer *where does this symbol sit*. Three more put a
factor on a comparable scale so several can be **combined**:

| Primitive | Returns | Description |
|---|---|---|
| `xs_zscore(<expr>)` | real | `(value − mean) / sd` across the universe, sample sd. |
| `xs_rank_pct(<expr>)` | real in [0, 1] | Percentile rank. Lowest = 0, highest = 1. |
| `xs_winsorize(<expr>, k)` | real | z-score clamped to ±`k` standard deviations. `k` may be fractional. |

You need these the moment a strategy scores on more than one factor.
Adding raw ranks weights each term by universe size; adding raw factor
values weights them by unit, so a $900 name's dollar volatility swamps
a momentum ratio. Standardize first and the weights mean what they say:

```qe
let score = factor("defensive_momentum",
  expr =  0.50 * xs_zscore(lag(close, 21) / lag(close, 252) - 1.0)
        + 0.20 * xs_zscore(lag(close, 21) / lag(close, 126) - 1.0)
        - 0.30 * xs_zscore(realized_vol(close, 60)))
```

`xs_rank_pct` is the outlier-tolerant alternative: one name at 100× the
rest compresses every z-score toward zero, and leaves percentiles
untouched. `xs_winsorize(x, 2.5)` is the middle ground — keep the
cardinal spacing, cap the tail.

Defined behaviour at the edges, all of it deliberate:

- **NaN members are excluded from the statistics**, not counted as zero.
  A symbol still warming up gets NaN itself, and every other symbol's
  value is exactly what it would be if that symbol were not in the
  universe. `xs_rank_pct`'s denominator counts only warm members, so
  the warm names still span the full [0, 1].
- **An infinite member is excluded on the same terms as a NaN one.**
  It gets NaN itself and no allocation, and everyone else scores as if
  it were not in the universe. Worth knowing because infinities are not
  hypothetical here: `lag(close, 21) / lag(close, 252) - 1.0` divides by
  a lagged close, and a CSV with an empty price field loads that bar's
  close as `0.0`. Before this was handled, one such name made the mean
  infinite and the sd NaN, which the code below read as "no dispersion"
  — every symbol in the universe got `0.0`, indistinguishable from a
  genuinely flat factor.
- **Fewer than 3 finite members → NaN for everyone.** A sample sd over
  two points is half the gap between them, so every z-score would be
  ±0.7071 regardless of the data: the sign and nothing else, dressed up
  as a normalized factor.
- **Zero variance → 0.0, not NaN.** Every name scored the same, so every
  name is exactly average. A NaN here would poison a weighted composite
  and drop the symbol from selection for a reason unrelated to its
  merit. Near-zero variance (sd < 1e-12) takes the same branch, because
  dividing by 1e-15 turns float noise into z-scores of 1e12.
- **Ties are deterministic**, broken by symbol index, so the same
  universe produces the same answer every run.

All three run in the **same single pre-pass** as the selection
primitives: each site's inner expression is evaluated exactly once per
symbol per timestamp. Nesting any cross-sectional call inside another
is rejected at bind time, normalizers included — `xs_zscore(rank(x))`
and `is_top(xs_zscore(x), 10)` are both errors, because the pre-pass
would need the outer site's results before it could run the inner.

Backtest and live use the same code path, so a composite score produces
identical values in both.

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
  The parallel **sizing** budgets (`budget_weights`, see
  [leg sizing](#leg-sizing-budget-sized-by-default-epic-83)) compose
  as a plain product of the slot weight and the inner budget weight,
  *without* the renormalization the accounting weights apply — that
  divide would undo exactly the `1/k` → `1/n` deflation EPIC-83
  removed. So the 4-leg `signalize_universe(top_k = 2, bottom_k = 0)`
  slot at `weights = [0.5, 0.5]` below carries accounting weights
  `[0.5, 0.125, 0.125, 0.125, 0.125]` and budgets
  `[0.5, 0.25, 0.25, 0.25, 0.25]`.
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
  See [`live-trading-runbook.md`](live-trading-runbook.md) for
  the picking-limits heuristic.

  **All four are POSITIVE magnitudes, and `<= 0` disables the
  gate.** Since EPIC-83 the three USD knobs are validated at load
  time: a negative value is a hard config error naming the knob and
  the positive spelling to use. `risk_max_position_per_symbol` is
  **USD notional** (`|projected position| × latest mark`), not a
  share count — the pre-2026-06-11 code compared raw shares, which
  let a 1-share $1,000 order sail under a cap of `150` that the
  deploy configs had already written as dollars.

  > `risk_max_daily_loss_usd` used to be documented as a NEGATIVE
  > number while `qe::net::SafeBrokerLimits::max_daily_loss` — the
  > backstop one layer below — read it as positive. The same value
  > therefore armed one layer and silently disabled the other. The
  > convention is now positive everywhere: write `1500`, not
  > `-1500`, for a $1,500 max daily drawdown. A hand-built
  > `RiskLimits` (not reachable from `.qe`) that still carries a
  > negative is normalized to its magnitude and logged at
  > `error` — failing **armed** is the safe direction. Hitting the
  > threshold trips the kill-switch for the rest of the session, and
  > the kill-switch is one-way: clearing it means
  > `qe_daemon stop && qe_daemon start`.
  >
  > A trip means "no more orders go out". It does **not** cancel
  > orders already working at the broker, does not flatten your
  > book, and does not stop the daemon — those orders can still
  > fill after the trip and will be booked normally. Cancel them
  > with `qe_daemon cancel-all`, the F6 **CANCEL ALL** control, or
  > by hand in TWS / IB Gateway / Client Portal. See
  > [Live-trading safety, Layer 3](live-trading-safety.md#layer-3-kill-switch).
  >
  > Three of the four knobs — position, exposure, order rate — are
  > checked **when an order is submitted**. `risk_max_daily_loss_usd`
  > is different, and changed in EPIC-88:
  >
  > - **It is evaluated continuously** (T88.4), on every mark
  >   update rather than only on an order attempt. Before EPIC-88 a
  >   breach could sit latent for the whole of a `rebalance = N`
  >   cycle and then fire on whatever the equity happened to be N
  >   sessions later; a position could bleed straight through the
  >   limit all session without tripping, because the limit only
  >   ever bit the next order and that order might never come.
  >   Tripping needs **two consecutive breaching marks**, so one bad
  >   print cannot fire a one-way switch. The submit-time test is
  >   still there as a backstop.
  > - **It measures the strategy, not the account** (T88.3 / T88.5).
  >   The threshold is compared against the strategy's own
  >   marked-to-market P&L, anchored on `execution(capital = ...)`.
  >   An account-scale anchor with a strategy-scale limit is not a
  >   conservative approximation, it is an incoherent one.
  > - **It refuses to start half-configured.** Arming
  >   `risk_max_daily_loss_usd` without a usable
  >   `execution(capital = ...)` is a startup error (exit 4), not a
  >   silent fallback. A limit that measures nothing is how the $25k
  >   paper deployment ran three weeks with no daily-loss backstop.
  > - A P&L reading of "the book cannot be valued right now" is
  >   **not** treated as zero loss: it neither trips nor clears an
  >   accumulated breach streak.
  >
  > `qe_daemon status` reports `daily_loss_state` — one of `absent`
  > / `inert` / `unanchored` / `armed` — alongside
  > `strategy_capital_usd` and the strategy P&L. The four states are
  > kept distinct on purpose: "no limit configured" and "limit
  > configured but measuring nothing" are very different situations
  > and used to render identically.

```qe
live(
  base   = base,
  broker = "ibkr-paper",
  symbols = ["SPY"],
  ibkr = ibkr_connection(host = "127.0.0.1", port = 7497, client_id = 1),
  risk_max_position_per_symbol = 20_000,   # USD notional per symbol
  risk_max_gross_exposure_usd  = 60_000,   # Σ |price · pos|
  risk_max_daily_loss_usd      = 1500,     # POSITIVE; trips the kill-switch
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
- `end` — ISO date. Optional *to the evaluator*, but **required in
  practice**: omitting it leaves the field empty and every consumer
  then hard-fails. `qe_factor` reports `universe.data:
  yahoo_template requires both 'start' and 'end' dates`; `qe_run`
  and the dashboard's sweep loader fail the same way.
  **The loader does not read the clock**; there is no
  "defaults to today" behaviour. Always write an explicit end date.
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
- **Pure**: `abs(x)`, `max(a, b)`, `min(a, b)`, `price_level(x)`.
- **Forecasting** (`predict_return`, `lag_return`,
  `rolling_vol`, `realized_vol`, `rolling_zscore`) — documented in
  [`forecasting.md`](forecasting.md). Read the units note on
  `rolling_vol` vs `realized_vol` under
  [Volatility: `rolling_vol` vs `realized_vol`](#volatility-rolling_vol-vs-realized_vol)
  before you use either.
- **Cross-sectional** — selection (`rank(x)`, `quantile(x, q)`,
  `is_top(x, k)`, `is_bottom(x, k)`) and normalization
  (`xs_zscore(x)`, `xs_rank_pct(x)`, `xs_winsorize(x, k)`). Only
  meaningful inside a universe; see
  [Cross-sectional primitives](#cross-sectional-primitives).

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
