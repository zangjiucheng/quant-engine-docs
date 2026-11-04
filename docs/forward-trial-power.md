# What a 60-session forward trial can and cannot decide

The deployment gate requires `min_forward_sessions = 60`. Sixty sessions
of paper trading is three calendar months of waiting, so it is worth
being precise about what that buys — because two quite different things
could be meant by it:

1. **Operational**: does the machinery work? Is an evaluation happening
   on every close, do fills reconcile against the broker, do fees land
   where the model put them, does a restart recover the book?
2. **Statistical**: is this strategy's edge real?

These need wildly different sample sizes. Conflating them is how a trial
gets read as evidence for a claim it cannot support.

**The answer, stated plainly: a 60-session trial detects operational
failure. It cannot resolve a Sharpe.** Everything below is the
measurement behind that sentence.

Reproduce it with:

```bash
qe_run backtests/cross_sectional/cs_it_lowvol_rb20_equaluniverse_friction.qe -o strategy.json
qe_run backtests/cross_sectional/buy_and_hold_it_equalweight.qe -o benchmark.json
python3 -I tools/qe-forward-power.py --strategy strategy.json --benchmark benchmark.json
```

The benchmark is Gate 4's second comparison object — the same 30-name
equal-weight basket, never rebalanced. Both runs are at realistic
friction, which is what Gate 4 requires.

## The statistical half

Measured over 2 515 aligned sessions and 2 456 overlapping 60-session
windows, with no distributional assumption — this is what the estimator
actually did on the realised series:

| | |
|---|---|
| Full-sample Sharpe, strategy | **+1.267** |
| Full-sample Sharpe, benchmark | +1.116 |
| The advantage the argument rests on | **+0.151** |
| Standard deviation of the 60-session Sharpe | **1.848** |
| P(a 60-session Sharpe is negative) | **19.3%** |
| P(the 60-session advantage has the full sample's sign) | **49.6%** |

Read the last two rows together. **One window in five reports a negative
Sharpe for a strategy whose ten-year Sharpe is +1.27**, and getting the
*sign* of the advantage right is a coin flip. A 60-session result is
consistent with almost anything.

The analytic view agrees. Lo (2002), *The Statistics of Sharpe Ratios*,
eq. 9: for iid returns the per-period estimator has standard error

    SE(Ŝ) = sqrt( (1 + Ŝ² / 2) / W )

and the annualised estimator's is `sqrt(252)` times that. At W = 60 that
gives **±2.053** on an annualised Sharpe. Inverting it for the sample
size that would shrink the standard error to the +0.151 advantage:

    W = 252 · (1 + Ŝ² / 2) / 0.151²  =  11 098 sessions  =  **44 years**

The iid assumption is generous. Autocorrelation and volatility
clustering both inflate the true standard error, so 44 years is a
**floor**, not an estimate.

### On the earlier figures

EPIC-100's G-8 row quotes sd 1.825, P(Sharpe < 0) 20.3%, sign agreement
50.9%, SE ±2.696 and 6 165 sessions, from a `power.py` that lived in a
measurement scratch and no longer exists. The three empirical statistics
reproduce here within about a point. The two analytic ones do not, and
since that script cannot be re-run the discrepancy cannot be traced —
which is why `tools/qe-forward-power.py` writes its formula out in full.
The conclusion is unchanged and the rebuilt figure is the more
pessimistic of the two.

## The operational half, and a contradiction it exposed

The gate also wanted ten completed round trips. Measured on the same
run: **291 round trips over 2 516 sessions**, or 0.116 per session.
Across 2 457 overlapping 60-session windows:

| min | p10 | median | mean | p90 | max |
|---|---|---|---|---|---|
| 3 | 4 | **7** | 7.02 | 10 | 11 |

| bar | share of windows reaching it |
|---|---|
| 3 | **100.0%** |
| 5 | 88.5% |
| 7 | 57.6% |
| 10 | **18.7%** |

So the gate held `min_forward_sessions = 60` and `round_trips >= 10` at
the same time, and that pair **refuses four operationally sound trials
out of five**. Two numbers in one file contradicting each other.

The bar is now `min_round_trips` in the policy, set to **3** — the
largest value that refuses none of the 2 457 measured windows. It is a
policy field rather than a literal in `gate.py` precisely so that
raising it is an informed choice against the table above, and
`validate_policy` refuses any bar the declared window cannot deliver at
the measured rate.

## What this means for how the trial is read

- **A clean 60-session trial is evidence the system works.** Every
  close evaluated, every fill reconciled, fees where the model put them,
  restarts recovering — those are observable in sixty sessions, and
  `tools/qe-forward-receipt.py` measures them from the daemon's own
  journal.
- **A profitable 60-session trial is not evidence the strategy works**,
  and an unprofitable one is not evidence it does not. One window in
  five of a genuinely good strategy looks bad.
- **The gate's job is the first of those.** Its forward conditions —
  session continuity, reconciliation, incidents, round trips, realized
  cost — are all operational. None of them is a test of edge, and none
  should be presented as one.
- **Waiting longer does not fix this.** Resolving the advantage needs
  decades, so the decision to deploy rests on the research, on cost
  realism, and on the operational evidence — never on the trial's P&L.
