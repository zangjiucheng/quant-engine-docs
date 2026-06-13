# qe-dsl — expression DSL grammar (v1)

This is the **signal-layer** DSL — the per-bar expression language
that powers `signal(entry = ..., exit = ...)` inside a `.qe` file
and the `expr = ...` argument of `factor(...)` in research configs.

For the **config-layer** language wrapped around these expressions
(let-bindings, kwarg calls, `backtest` / `sweep` / `walk_forward`
/ `research`, ranges, arrays), see
[`qe-language.md`](qe-language.md). The two layers share a parser
but evaluate in distinct contexts; `qe-language.md` § "Two contexts:
**config** vs **signal**" spells out where each one applies.

The language is **pure expression** — no statements, no control flow,
no variable binding. Each `entry` / `exit` field in a `signal(...)`
value is one expression; the engine evaluates it once per bar.

## EBNF

```
expression  = or_expr ;

or_expr     = and_expr , { "or" , and_expr } ;
and_expr    = not_expr , { "and" , not_expr } ;
not_expr    = [ "not" ] , compare ;

compare     = sum , [ cmp_op , sum ] ;
cmp_op      = ">" | ">=" | "<" | "<=" | "==" | "!=" ;

sum         = product , { ( "+" | "-" ) , product } ;
product     = unary   , { ( "*" | "/" ) , unary } ;

unary       = [ "-" ] , primary ;

primary     = number
            | bool_lit
            | ident
            | call
            | "(" , expression , ")"
            ;

call        = ident , "(" , [ arg_list ] , ")" ;
arg_list    = expression , { "," , expression } ;

bool_lit    = "true" | "false" ;
ident       = letter , { letter | digit | "_" } ;
number      = digit , { digit } , [ "." , digit , { digit } ] ;
letter      = "A".."Z" | "a".."z" | "_" ;
digit       = "0".."9" ;
```

Whitespace (space, tab, newline) and line comments (`# ...` to end-of-line)
are skipped by the lexer.

## Precedence (low → high)

```
or                       — left assoc
and                      — left assoc
not                      — prefix unary
==  !=  <  <=  >  >=     — non-assoc, single comparison only
+  -                     — left assoc (binary)
*  /                     — left assoc
unary -                  — prefix
function call, ( ... )   — highest
```

Comparison is **non-associative** by design: `a < b < c` is a syntax
error, force the user to write `a < b and b < c`.

## Types

Every value is a `double`. Booleans are represented as `0.0` / `1.0`;
comparisons and `and / or / not` produce `0.0` or `1.0`.
`if` / ternary do not exist in v1 — combine with `and / or` or use
`max(a, b)` / `min(a, b)` to express conditional numerics.

## Variables (per-bar bindings)

| name | meaning |
|---|---|
| `close`      | bar `i` close                        |
| `open`       | bar `i` open                         |
| `high`       | bar `i` high                         |
| `low`        | bar `i` low                          |
| `volume`     | bar `i` volume                       |
| `bar_index`  | integer position of bar in series    |

All numeric. `bar_index` is `double` for uniformity but is exact for
indices < 2^53.

## Built-in functions

Stateful (per-call-site state slot allocated at parse time, reset on
strategy start):

| call                  | semantics                                                                 |
|-----------------------|--------------------------------------------------------------------------|
| `sma(x, n)`           | `qe::indicators::Sma(n)` fed `x` per bar — see `sma.hpp`                  |
| `ema(x, n)`           | `qe::indicators::Ema(n)` fed `x` per bar — SMA-seeded warmup              |
| `rsi(x, n)`           | `qe::indicators::Rsi(n)` fed `x` per bar — Wilder smoothing               |
| `cross_above(a, b)`   | true iff `a` strictly crossed above `b` this bar                          |
| `cross_below(a, b)`   | true iff `a` strictly crossed below `b` this bar                          |

Pure (stateless):

| call           | semantics                |
|----------------|--------------------------|
| `abs(x)`       | `std::fabs(x)`           |
| `max(a, b)`    | `std::fmax(a, b)`        |
| `min(a, b)`    | `std::fmin(a, b)`        |

The second arg to `sma` / `ema` / `rsi` must be a **constant integer
literal** — windows are baked at parse time so the underlying indicator
can be constructed once. Inside a `.qe` file, however, a top-level
`let fast = 10` works: the config evaluator inlines integer let
bindings into signal subtrees before this layer sees them. So
`sma(close, fast)` is a parse error in raw signal-DSL but parses
fine inside the `signal(...)` arg of `qe-language.md`. See
[`qe-language.md` § "Two contexts"](qe-language.md#two-contexts-config-vs-signal).

Forecasting builtins (`predict_return`, `lag_return`,
`rolling_vol`, `rolling_zscore`) and feature primitives live in
[`forecasting.md`](forecasting.md) — same per-bar evaluator, but
documented alongside the rolling-ridge math they exist for.

`cross_above` / `cross_below` keep one bar of history per call site
(the prior `a - b` sign).

## NaN handling

Indicators emit NaN until their window has filled. Any NaN propagates
through arithmetic and comparisons (per IEEE 754 — `NaN > x` is false,
`NaN != NaN` is true). The strategy state machine treats NaN entry /
exit signals as false (no trade).

## min_history inference

`qe::dsl::min_history(ast, env)` walks the AST and returns the maximum
warmup that any reachable indicator call requires. Nested calls
accumulate: `ema(rsi(close, 14), 21)` requires `14 + 21 = 35` bars
because the inner RSI emits its first non-NaN at bar 14 and the outer
EMA needs 21 of those.

## Errors

The lexer and parser surface errors as `qe::Result<T>` carrying
`qe::ErrorCode::InvalidConfig` with a message that includes a 1-based
column number, e.g. `expression: column 14: expected ')'`.

## Examples

```
# Classic SMA crossover
entry: "cross_above(sma(close, 20), sma(close, 50))"
exit:  "cross_below(sma(close, 20), sma(close, 50))"

# Trend + momentum
entry: "sma(close, 50) > sma(close, 200) and rsi(close, 14) < 30"
exit:  "rsi(close, 14) > 70"

# Volatility breakout (close above 20-day high mean + 2× range)
entry: "close > sma(high, 20) + 2 * (sma(high, 20) - sma(low, 20))"
exit:  "close < sma(close, 10)"
```

## Out of scope (v2 candidates)

- Historical indexing: `close[1]`, `close[i]`
- Let-bindings: `let s = ema(close, 12); s > sma(close, 50)`
- User-defined functions
- Short / pyramiding / stop / take-profit signals (state machine
  stays flat / long only in v1)
- Cross-symbol expressions for pairs trading
- String literals (none of the v1 builtins need them)
