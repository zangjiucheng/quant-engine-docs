# The deployment gate

Before a first real-money book exists, something has to decide that it
may. That decision is `python/qe_deploy/` — an offline evidence gate
that reads signed local receipts and reports `PASS` or `BLOCK`.

Its own docstring is the shortest statement of what it is for:

> A readiness report is neither a broker credential nor a profitability
> certificate. Receipts are local, reviewed attestations, verified by
> content hash, identity and age. They are not authenticated broker
> statements. Unknown facts block launch. The native daemon retains its
> independent account, sizing and pre-trade gates.

Four things follow from that, and each one is load-bearing:

- **It contacts nothing.** The report carries `orders_sent: 0` and
  `broker_contacted: false` as fields, not as prose. It starts no
  service, fetches no price and sends no order.
- **It is not the last line of defense, it is the first.** Passing it
  changes nothing about the daemon's own runtime controls — see the
  [safety model](live-trading-safety.md)'s six layers, all of which
  still apply to every order afterwards.
- **A missing fact is a blocker, not a warning.** Every check that
  raises — including one that raises `KeyError` because a field is
  simply absent — is recorded as `passed: false`, and one failed check
  makes the decision `BLOCK`.
- **It cannot certify that a strategy makes money.** It checks that the
  evidence for a decision exists, is internally consistent, and was
  produced before the outcome it is used to justify.

## Running it

```bash
python3 -m qe_deploy \
    --policy   policy.json \
    --evidence evidence.json \
    --native   native_validate.json \
    --expected pinned_identities.json
```

Exit status is `0` on `PASS` and `2` on `BLOCK`. The report goes to
stdout as JSON: a `decision`, then one entry per check with its `id`,
`passed` and a `detail` string that names what was wrong.

**Omitting `--expected` does not relax the gate, it fails it.** The CLI
appends a failing `expected_identity` check and forces `BLOCK`, because
an offline report cannot establish an independently pinned identity by
leaving the argument off.

For a live instance already declared in a fleet manifest, the collector
wraps the same evaluation:

```bash
scripts/qe-strategy-fleet.py gate --manifest fleet.json --id <instance>
```

That subcommand refuses an `--id` that is not a declared **live**
instance. `check` stays static-only.

## The nine checks

| check | what it demands |
|---|---|
| `policy` | The whole policy document is internally consistent: at most USD 5 000, `CASH`, zero interest, whole shares, no shorting, risk caps that preserve the cash reserve and do not clip the frozen allocation, a phase frozen before outcomes, ≥ 60 forward sessions, strict freshness and spread bounds. |
| `identity` | The evidence, the native config and the independently pinned hashes all name the same profile, config, binary, strategy and `U…` account. The native config must declare `ibkr-live`, carry the same risk limits as the reviewed profile, and have validated with `broker_contacted: false`. |
| `small_capital_research` | A USD 5 000 whole-share non-compounding zero-interest rerun of the full frozen study — all 20 phases under base *and* stress costs, 40 rows, nothing selected. Plus a positive reported turnover, point-in-time universe membership dated no later than the training start, and an explicitly named price basis. |
| `prospective` | A forward trial with no holes: every session between inception and the frontier, each complete, reconciled, fee-complete and settled, with ≥ 2 executed rebalances, the policy's round-trip count, no unresolved incidents, no external cash flows, and a **measured** realized slippage no worse than the stress scenario the research cleared. |
| `operations` | Six operational tests actually ran, the test report and rollback release are pinned by hash, the receipt is under seven days old, and nothing else writes to the account. |
| `launch_window` | The launch session was committed before its preceding close, the forward evidence is complete through that preceding session, and it is now 09:00–09:25 New York on that precommitted day. No late catch-up. |
| `broker_snapshot` | A fresh snapshot of the right `CASH` USD account, flat, with no working orders, connected, reconciled, and settled cash at least the policy capital. |
| `startup_plan` | A first-entry rehearsal: warmup bars complete through the preceding session for every native symbol, live (type 1) quotes inside the freshness and spread bounds, whole-share quantities that match the native rehearsal exactly, weights summing to the frozen gross, and a modelled spend that neither consumes the cash reserve nor finances buys from unsettled proceeds. |
| `operator_review` | A human signed off, on *this* evidence bundle, within the last 24 hours. |

## The six receipts

Five receipts are bound into the signoff; the sixth carries the binding
and therefore cannot be inside it.

| receipt | produced by | signed into the digest |
|---|---|---|
| `research` | `tools/qe-research-receipt.py` | yes |
| `forward` | `tools/qe-forward-receipt.py` | yes |
| `operations` | `tools/qe-operations-receipt.py` | yes |
| `broker` | `tools/qe-broker-receipt.py` | yes |
| `plan` | the native startup rehearsal | yes |
| `approval` | the operator, with `tools/qe-approval-digest.py` | **no — it holds the digest** |

`review_digest` is a SHA-256 over the binding, the launch session, the
launch commit time and the content hash of each of the five signed
receipts. Change any one of them and the approval stops matching, which
is the point: a signoff is a statement about a specific bundle, not a
standing permission.

## What the producers cannot measure, and say so

A receipt producer that fills in every field looks better and is worth
less. These name their gaps instead.

**`broker` — 6 of 9 fields.** The daemon's control socket is read-only
and refuses the `account` verb by name, so three fields are emitted
under `not_measured` with the reason:

| field | why not |
|---|---|
| `account_type` | `CASH` versus `MARGIN` comes from the broker's account description, which is not on the read-only channel. |
| `announced_accounts` | Same refusal — the Gateway's announced account list is not served. |
| `settled_cash_usd` | The channel offers an account cash *balance* and the ledger's `cash_usd`. The gate's own refusal text says NAV and buying power are not settled cash, so substituting one is how the field stops meaning anything. |

The producer does report what *refutes* a claim it cannot confirm:
buying power above 1.0× equity is impossible in a cash account, so the
measured ratio refutes `account_type: "CASH"` without reading the field.

**`operations` — 5 of 6 booleans.** It runs the tests rather than
trusting a tick, and it refuses `settlement_cash_guard_tested` because
there is no guard to test: the daemon implements no notion of settled
cash. That matters more than one flag, because `settled_cash_usd` bounds
two separate checks. `no_other_account_writer` is also left unmeasured —
it is a claim about other processes, other machines and a human in TWS,
and no test inside this tree can see any of that.

**`approval` — the tool refuses to produce it.**
`tools/qe-approval-digest.py` computes the digest an operator signs and
will not emit `approved: true`. A program that can sign its own
authorisation is not an authorisation.

## Three numbers in the policy that are measurements

**Round trips (`min_round_trips`).** A policy may not demand more
completed round trips than the window it also declares can deliver. The
rate is measured — 291 round trips over 2 516 sessions on the
strategy's own ten-year run at realistic friction — and the gate
refuses a policy that asks for more than
`min_forward_sessions × 291/2516`. The earlier hard-coded pair
(60 sessions, 10 round trips) refused four operationally sound trials
out of five. Derivation and distribution:
[forward trial power](forward-trial-power.md).

**Stress slippage (≥ 22 bps).** The stress cost floor was 8 bps while
this book's measured realized slippage was
[22.07 bps](safety-and-limitations.md#what-execution-actually-cost)
notional-weighted over 89 executions. A stress scenario cheaper than
what already happened is not a stress scenario. The 22 is a static
backstop for a policy validated with no forward evidence in hand; the
live bound is in the forward check, where the measured number must not
exceed the stress scenario the research cleared.

**The drawdown branch (× 0.60).** The workspace's Gate 4 passes a
strategy on Sharpe *or* on a drawdown at most 60 % of the benchmark's.
In this strategy family drawdown is very nearly a measurement of how
much is deployed — fitted at 0.679 × deployed fraction with r = +0.977,
while Sharpe moves 0.24 across a 6.7× change in deployment, and in the
wrong direction. So the gate adds no threshold of its own and applies
the existing 60 % to a drawdown *normalised by deployment*. A book
cannot earn the discount by simply holding less.

## Two acceptance scopes, because a one-year holder is not a 17-year one

`acceptance_scope` picks which question the research evidence has to
answer. The two are not a strict/loose pair — they are different
statistics.

| scope | targets | evaluated over |
|---|---|---|
| `research` | `net_cagr` ≥ 12%, `net_sharpe` ≥ 1.0, `max_drawdown` < 20% | p50 across the 20 frozen phases |
| `personal_1y` | `net_cagr` ≥ 8%, `median_1y_return` ≥ 8%, `dd_breach_free_rate` ≥ 95% at a `dd_cap` of 20% | one-year windows of the equity curve |

The `research` targets cannot be relaxed. The `personal_1y` thresholds
are floors — a stricter policy is accepted, a looser one refused, each
of the four checked separately.

**Three reads, and all three must pass.** A `personal_1y` research
receipt reports `one_year_reads` for an `overlapping`, a `disjoint` and
a `calendar` read, and the gate requires every one of them. This is the
rule that does the work: on the project's own candidates one rule
cleared the overlapping read at a median 1-year return of +9.48% and
failed the disjoint read at +7.93%. A gate that took the best read — or
one the receipt got to nominate — would accept exactly what the
operator's gate rejects. Omitting a read is refused by name, not
silently treated as a narrower claim.

**The drawdown criterion is a rate, and the rate is a point estimate.**
`dd_breach_free_rate` is the measured fraction of one-year windows whose
maximum drawdown stayed under `dd_cap`. It is **not** a probability
about any future year and the gate's own output says so: seventeen
independent breach-free years put the 95%-confidence lower bound on the
true rate at 83.8%, and establishing 95% at 95% confidence would need 59
of them. Read it as an acceptance threshold on a measured window, on the
same footing as every other number here.

**Why CAGR survives beside a one-year test.** The median one-year return
of a volatile series runs *above* its compound rate, and the gap is not
small: one candidate shows +9.48% median against a 7.80% full-window
CAGR. A strategy is held year after year, so what accumulates is the
compound rate — and the candidate the median-only form admits is the one
that fails the median test itself on the disjoint read. Both tests, not
one.

**A third scope used to exist and is now refused by name.** `personal`
gated on a daily Sharpe floor. At a one-year horizon that floor does not
discriminate — a 0.156 spread in full-window Sharpe produces a
1.2-point spread in the one-year pass rate — so it was superseded rather
than retuned. A policy still carrying it gets told which record
superseded it and what to use instead, because deleting the name would
have produced "unknown acceptance scope" and sent the reader to the
wrong question.

## Two schedule shapes, because a month-end rule has no phase

A **phase** is `eval_index % rebalance_bars` — which day of the
rebalance cycle a run happens to start on. It is an arbitrary nuisance
parameter, so a `rebalance_offset` rule reports all 20 of them under
both cost modes and the acceptance metrics are the p50 across them. That
is what stops anyone picking the lucky start day.

A month-end rule has no such offset: the calendar pins the date. So
`schedule_kind: "month_end"` replaces the 20 phases with **two**
requirements, which answer different questions and do not substitute for
each other.

**Marginalisation — the month-end day is shifted.** `month_end_offsets`
must be contiguous from 0 and at least **5** long, giving one `phases`
row per (offset, cost) for the deployed variant. Every acceptance metric
is then the p50 across offsets, exactly as with the 20 phases. The
one-year reads carry `by_offset` and **the gate computes the median
itself** rather than trusting a reported aggregate — a hand-typed median
is the one field a receipt could get wrong in its own favour, and
picking an offset is what this exists to prevent.

The 5 is a judgement and the only number here that is not a
measurement. A month is about 21 sessions, but shifting a month-end rule
across a whole month stops it being a month-end rule; the last five
sessions still are one, and five is enough for a median to mean
something. Fewer would let one unlucky offset be outvoted by two lucky
ones.

**Disclosure — every pre-registered variant is reported.**
`pre_registered_variants`, `variant_summaries`, a named
`deployed_variant` and a named `reference_variant`. The deployed variant
must pass on its own; the siblings only have to be *there*.

That asymmetry is deliberate. Four variants are four strategies, not
four views of one, so a cross-variant median would be nobody's number
and a sibling failing the bar is information rather than a veto. What
this refuses is a receipt carrying only the rule that won. The reference
is called out separately because it is the line that could show the rule
was unnecessary — a receipt without it is not a comparison, so its
absence is refused by name.

## Turnover, survivorship and price basis

Three facts the protocol required the research receipt to report and
nothing checked:

- **Turnover** must be positive in the research receipt, and the forward
  trial's measured annualised turnover must not exceed it. Research that
  assumed less trading than actually happened has understated costs, and
  the direction is unambiguous, so no tolerance softens it.
- **Survivorship**: `point_in_time` must be true and the universe
  selection date no later than the training start. A universe chosen as
  of a later date was selected with information the training period did
  not have.
- **Price basis** must be named explicitly, `engine` or `nominal`. The
  vendor's close is adjusted for *future* splits, which is look-ahead for
  any factor ranking on a price level — see
  [safety & known limitations](safety-and-limitations.md).

## What a `BLOCK` is

A refusal with a reason. Each failed check carries the text that
explains it, and the texts are written to be acted on rather than
decoded. `BLOCK` is also the default: an empty bundle, an unreadable
file, a missing argument and a stale approval all reach it, because the
alternative — defaulting to permission when evidence is absent — is the
failure this whole mechanism exists to prevent.

---

## See also

- [Safety model](live-trading-safety.md) — the six runtime layers that apply after this gate
- [Safety & known limitations](safety-and-limitations.md) — what execution actually cost, and the parity matrix
- [Forward trial power](forward-trial-power.md) — what 60 sessions can and cannot decide
- [Daemon runbook](live-trading-runbook.md) — what runs on the host
- [Strategy comparisons](strategy-comparison.md) — the manifest and the collector
