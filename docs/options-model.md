# Options pricing model — Black-Scholes-Merton, q = 0

This page pins down the model used by `qe::analytics::black_scholes`
— the math layer behind the F5 RISK · GREEKS panel. The goal is for
a reader to map the closed-form code to a textbook and understand
the assumptions the dashboard makes.

## Scope

Pricing + 5 Greeks (Δ Γ ν Θ ρ) for European-exercise calls and puts,
plus a Brent root-find for implied volatility.

Out of scope for this layer:

| Concern | Why excluded |
|---|---|
| American exercise premium | SPY options are American. BS European is the closed-form. Documented caveat: error < 1% for short-dated near-ATM, 3-5% for deep ITM puts. |
| Continuous dividend yield `q` | Assumed `q = 0`. SPY's ~1.4% yield warps long-dated Greeks by a few percent. Knob can land in v2. |
| Stochastic vol (Heston, SABR) | Out of scope. We solve a flat-vol IV per (strike, expiry). |
| Term-structure of risk-free rate | One scalar `r`. Default `0.05` (set in `DashboardConfig::risk_free_rate`). |
| Greeks of higher order (vanna, vomma) | Not surfaced in F5 RISK · Greeks. Easy to add later. |

## Notation

| Symbol | Meaning |
|---|---|
| `S` | Spot price of underlying (today) |
| `K` | Strike |
| `T` | Time to expiry, in **years** (365.25-day basis) |
| `r` | Continuously-compounded risk-free rate |
| `σ` | Annualized volatility (the IV we solve for) |
| `N(·)` | Standard normal CDF |
| `n(·)` | Standard normal PDF |

The two BS auxiliaries used everywhere below:

```
d1 = (ln(S/K) + (r + σ² / 2) · T) / (σ · √T)
d2 = d1 - σ · √T
```

## Closed-form pricing

### Call

```
C = S · N(d1) - K · e^(-r·T) · N(d2)
```

### Put

```
P = K · e^(-r·T) · N(-d2) - S · N(-d1)
```

(Equivalent put-call parity check: `C - P = S - K · e^(-r·T)`. Useful
for unit tests.)

## Greeks (analytic, no numerical differentiation)

All formulas assume `q = 0`. Theta is expressed **per calendar day**
(divide the per-year formula by 365.25) because that's what traders
quote and what the F5 RISK · Greeks table needs.

### Delta — ∂Price/∂S

```
Call:  Δ = N(d1)
Put:   Δ = N(d1) - 1
```

### Gamma — ∂²Price/∂S² (same for call and put)

```
Γ = n(d1) / (S · σ · √T)
```

### Vega — ∂Price/∂σ, per **1.00** of vol (not per 1%)

```
ν = S · n(d1) · √T
```

(F5 RISK · Greeks surfaces ν in $ per 1 vol-point — multiply by 0.01 for the
"per 1% IV move" convention some platforms use.)

### Theta — ∂Price/∂t, per **calendar day**

Per-year formulas (divide by 365.25 for the daily quote):

```
Call:  Θ_yr = -(S · n(d1) · σ) / (2·√T) - r · K · e^(-r·T) · N(d2)
Put:   Θ_yr = -(S · n(d1) · σ) / (2·√T) + r · K · e^(-r·T) · N(-d2)
```

Theta is **negative for long options** (time decay). Stored as the
per-day number directly.

### Rho — ∂Price/∂r, per **1.00** of rate (not per 1%)

```
Call:  ρ =  K · T · e^(-r·T) · N(d2)
Put:   ρ = -K · T · e^(-r·T) · N(-d2)
```

## Edge cases

The closed-form blows up or denormalizes at degenerate inputs.
`black_scholes::price` returns `NaN` (and Greeks return all-NaN) in
the cases below — the F5 RISK panel renders those as "—" rather than 0.

| Case | Why it breaks | Returned |
|---|---|---|
| `T <= 0` (expired) | `σ·√T = 0` → divide-by-zero in `d1` | intrinsic price + Δ = 0/1/−1 step, others 0 |
| `σ <= 0` | Same divide-by-zero | NaN price + Greeks |
| `S <= 0` or `K <= 0` | `ln(S/K)` undefined | NaN price + Greeks |

Expired-leg handling is special because the user can leave a stale
position file with a past expiry — we render intrinsic value and Δ at
{−1, 0, +1} so the position contributes to NAV correctly until they
edit the file.

## Implied volatility — Brent's method on `BS_call(σ) - mid`

We never trust Yahoo's `impliedVolatility` field. The reported field
is often stale (cached weekly) or missing entirely on illiquid strikes,
and it's pre-computed against Yahoo's spot which may be 30s old. We
re-solve every chain refresh:

```
mid = (bid + ask) / 2         (filter: bid > 0 ∧ ask > 0 ∧ ask < 2·bid)
σ*  = solve BS_call(S, K, T, r, σ) = mid  for σ ∈ [σ_lo, σ_hi]
```

If the mid filter rejects the leg (illiquid quote, stale), the
fallback chain is:

1. Use Yahoo's reported `impliedVolatility` if present and > 0
2. Use `lastPrice` instead of mid and retry the solver
3. Return `NaN` IV → Greeks render as "—"

### Solver

Brent's method (combination of bisection + inverse quadratic
interpolation) on a price-difference root finder. Implementation:

- Bracket: `σ_lo = 1e-4`, `σ_hi = 5.0` (corresponding to 0.01% and
  500% annualized — wider than any realistic equity IV)
- Tolerance: `1e-6` on σ (well below the precision of mid quotes)
- Max iterations: 100 (Brent converges quadratically near root; 100
  is a safety net, expected convergence is 10-15 iterations)
- If `f(σ_lo) · f(σ_hi) > 0` (no sign change) → return NaN, the leg
  is unsolvable

`std::lower_bound`-style: returns the σ such that `BS(σ) ≈ mid` within
tolerance, or `NaN` if no root in bracket.

### Why Brent and not Newton-Raphson

Newton would converge in 3-5 iterations using vega as the derivative —
faster per call. But Newton diverges near deep ITM / very short-dated
options where vega → 0, and we need bulletproof convergence on a
~200-strike chain refresh. Brent's hybrid bisection guarantee is
worth the extra iterations.

## References

- Hull, *Options, Futures, and Other Derivatives*, 10th ed., Ch. 15-17
  (BSM derivation + Greeks)
- Brent, *Algorithms for Minimization Without Derivatives* (1973),
  Ch. 4 — the root-find algorithm we use
- `QuantLib::BlackScholesCalculator` — reference values for the unit
  test fixture in `tests/test_black_scholes.cpp`
