# Migration and incident history

Version-specific migrations and dated incident evidence. These records
describe behaviour at the time; use [Safety and known limitations](safety-and-limitations.md) for current constraints and the
[daemon runbook](live-trading-runbook.md#upgrading-an-existing-deployment)
for the upgrade procedure.

## Breaking changes

### 0.4.0 — `qe_daemon` refuses a price-level volatility factor

**This is the one that will stop a running deployment from restarting.**

A daemon restarted on a 0.4.0-or-later build **refuses to start** on a
config that applies `rolling_vol` to a bare price variable. Both
`qe_daemon validate` and `qe_daemon start` exit **4**, before any socket
to the venue is opened. `qe_run` is unaffected and still loads such a
config, because reproducing a legacy result has to keep working.

Why it is fatal rather than loud: this warned at load time for a full
release, and a deployment carried the warning in its startup log every
session and ran on it for six weeks.

**Migration — do this before you restart on a new build.** Either:

```qe
// Before — refused at the live boundary
rolling_vol(close, 20)

// Option A: you meant return volatility. A DIFFERENT factor.
realized_vol(close, 20)

// Option B: you meant dollar dispersion. Exact identity, same numbers.
rolling_vol(price_level(close), 20)
```

- **Option B is a no-op.** `price_level(x)` returns `x` and allocates no
  state. The arithmetic is byte-for-byte unchanged, so results,
  rankings and fills are identical. You do not need to re-run your
  gates. Use this if dollar dispersion is genuinely what you want, and
  the config now says so on the page.
- **Option A is a different strategy.** `realized_vol` measures
  something else, will select different names, and needs its gates
  re-run from scratch. Do not treat it as a rename.
- **Option C**, if you need the daemon up right now:
  `--allow-price-level-vol`. It starts and prints the whole refusal at
  ERROR every session. Use it to buy time, not as a resolution.

Verify before you restart:

```bash
qe_daemon validate path/to/live_config.qe   # exit 0 = deployable
```

### 0.4.0 — a config whose warmup cannot be delivered refuses to start

Previously the daemon requested a fixed ~41 bars and started regardless
of what came back. It now derives the requirement from the config and
**refuses when any symbol is short**. See
[Current live warmup limits](safety-and-limitations.md#current-live-warmup-limits).

This is breaking in the operational sense: a config that started
yesterday can refuse today. It was not working yesterday — it was
trading on NaN-sorted rankings — but the failure is now visible where
it used to be silent. A recent listing in your universe is the most
likely trigger.

### 0.4.0 — reconciliation drift blocks submissions

Drift that previously produced a WARN and continued trading now stops
all new submissions, entries **and exits**, until acknowledged. See
[Reconciliation and restart behaviour](safety-and-limitations.md#reconciliation-and-restart-behaviour).
A supervisor script that assumed a daemon reporting healthy is a daemon
that will trade needs to check `qe_daemon reconcile` (exit 3 while
blocked) or the `reconcile_blocked` field on `status`.

### 0.4.0 — `results.json` portfolio writer is `schema_version` 6

Was 4; 5 was already taken by walk-forward, since schema numbers are one
namespace across all four writers rather than a per-writer counter. The
only difference is the added `provenance` block. Every v4 field is
emitted unchanged with the same meaning, so a reader that accepted v4
accepts v6. The dashboard reads both, and degrades a malformed
`provenance` block to "absent" rather than failing the load.

### 0.4.0 — `win_rate` and `avg_win_loss` are `null`, not `0.0`

On portfolio, multi-symbol and multi-strategy runs these two metrics are
not computed — round-trip trades do not pair cleanly across legs — and
they used to serialize as `0.0` and print as `Win rate 0%`. They are now
`null` in JSON and `—` on the console, and `provenance.not_computed`
names the metric and the reason.

If you have a consumer that reads these fields, it must handle `null`.
Multi-symbol **walk-forward** was one of the paths left reporting
`0.0`, because that value fed the parameter-selection argmax; the fix
changed the argmax and the value together, so it reports `null` there
too. A sweep over a portfolio was the other, and was fixed in v0.4.2.
See
[When the metric cannot rank anything](walk-forward.md#when-the-metric-cannot-rank-anything)
for what an optimizing run now does when the metric produces nothing —
including the new per-window `best_score`, and the refusal of
`metric = "win_rate"` on a multi-symbol universe.

### 0.4.0 — `Strategy::min_history()` removed (C++ API)

A pure virtual on the strategy-runner interface plus CRTP hooks on both
strategy bases. Nothing in the engine, daemon, reports or live path ever
called it — warmup has always been enforced by NaN propagation — and a
warmup number nobody reads is worse than none, because it reads as a
guarantee. The DSL-level `qe::dsl::min_history()` stays and is now what
sizes the live warmup. Only affects code embedding the C++ library.

---

## Versioned incident notes

Dated by the period the defect was live, oldest first — which is not
the order they were found in; the first two were both uncovered on
2026-08-01, weeks after they started. Each says what was wrong, how it
was found, and what changed.

### 2026-06 to 2026-07-31 — six weeks on the wrong factor

**What happened.** A paper deployment ran a strategy whose selection
factor was `rolling_vol(close, 20) * -1.0`, described in its own config
and every downstream note as a low-volatility factor. It is not one; see
[Semantic traps](safety-and-limitations.md#semantic-traps). It is approximately a low
share-price tilt. Every selection the deployment made in those six
weeks was made on the wrong measurement.

**How it was found.** Not by an alert. By a factor-audit pass that
measured what the function returns and compared it to what the name
claims.

**Why nothing stopped it.** The load-time lint fired on every single
startup, into a log nobody was reading.

**What is important about it.** The recorded gate results for that
strategy are real measurements *of that factor*. They were not
retracted. What was wrong was the name, and every sentence downstream
that reasoned from the name.

**What changed.** The lint became a hard refusal at the live boundary
(exit 4), and the finding now travels with the artifact — into
`provenance.warnings`, onto stdout in the run summary, and into a banner
above the KPI cards in the HTML report. A warning that only exists in a
log is not a control.

**A related fact, kept separate on purpose.** That deployment's own
journal records no confirmed fills at all in those six weeks. So "traded
the wrong factor for six weeks" is accurate about what the strategy
selected.

> **Correction, 2026-08-04.** This paragraph continued "and wrong about
> what the account ever held", inferring an empty account from the
> journal's silence. That inference is withdrawn. The journal fact
> stands — it records no confirmed fills — but the broker's book
> contains holdings that appear to predate the 07-31 session, and a
> journal that never saw a fill is not evidence that none occurred.
> Their origin is an open question, not a settled one.

### 2026-07-31 — ten fills reported as booked against a flat account

> **Correction, 2026-08-04. The headline claim on this section was
> wrong, and the error is more instructive than the incident.** The
> account was not flat. Every piece of evidence that said it was came
> from two broker queries that are now known to return a confident
> empty answer regardless of the truth — see "an empty answer that
> was not an empty book" below. The daemon's ten fills were real.
> The section is kept, retitled, because the reasoning failure it
> now documents is worth more than the incident it originally did.

**What was believed.** Between 09:30:31 and 09:30:33 the daemon wrote
ten `order_fill` events and ten `position` events — its first ever. Its
own reconcilers appeared to contradict them within ninety seconds: nine
projection drifts at 09:31:04 ("gate held 4, broker holds 0 with 0 still
in flight"), ten position drifts at 09:32:04 (`booked=N broker=0
drift=-N`). A direct account query the next day returned no positions
and no working orders. That was read as confirmation.

**What was actually true.** Every `broker=0` in those reports is the
same defect: `decode_position_data` was reading a field POSITION_DATA
does not contain, so every row failed to parse, and the frame handler
dropped failures silently. `list_positions()` returned success with an
empty vector. The reconcilers were not contradicting the fills — they
were comparing the journal against nothing and rendering the nothing as
zero. The follow-up "no working orders" is worthless for a second,
unrelated reason: that query is client-scoped and was issued from a
different client id than the one that placed the orders.

The fills were real. The broker's book holds seven of those symbols at
exactly the journal's quantities, and for each one the broker's average
cost exceeds the journal's fill price by exactly the $1.00 per-order
commission — arithmetic that cannot arise from positions that were never
opened.

**What is still unexplained.** One symbol recorded **both** an
`order_fill` and an `order_rejected` in the same second — broker code
2161, the price-cap notice, whose text ends "you will not receive a
fill". One order cannot be both, and the decoder defect does not account
for it.

**The lesson worth keeping.** Three independent controls agreed the
account was flat, and all three were reading the same broken sensor. The
agreement felt like corroboration and was not: a defect upstream of
every consumer produces unanimity, not contradiction. Nothing in the
system distinguished "the venue says zero" from "we failed to parse what
the venue said", because the code path that discarded the failure was a
bare `return`.

**Root cause.** The connection synthesised fills from order-status
cumulative-fill transitions and discarded the execution-details stream
entirely, on the reasoning that the two carry the same facts. They do
not: order status is the order's state, execution details is the broker
saying shares changed hands. Discarding one leaves nothing to disagree
with on the day they differ, and the disagreement is the whole signal.

**What changed.** Executions are decoded, logged with their execution
id, exchange and account, and counted per order (0.3.3). A shortfall
between what the order claims and what the executions account for is now
**durable state**, not a log line: it is counted, exposed on `status`,
flips the reconcile verdict off "clean", and raises a submission block
(0.4.0).

That work still stands on its own reasoning — order status is the
order's state, execution details is the broker saying shares changed
hands, and discarding one leaves nothing to disagree with on the day
they differ. But it should be read as a deliberate tightening, not as a
remedy for an observed loss, because the loss did not occur. Note also
that the tiebreaker named in the original version of this entry — "the
broker's own position, which the reconciler queries" — was for the whole
of that period a query that always answered zero.

### 2026-08-04 — an empty answer that was not an empty book

**What happened.** `decode_position_data` read a `primaryExchange` field
that IBKR's POSITION_DATA frame does not carry. Every subsequent field
shifted by one, so `position` and `avgCost` were parsed from
non-numeric text, every row failed, and the frame handler discarded each
failure with a bare `return` — no log line, no counter. `POSITION_END`
then sealed the empty map and `list_positions()` returned **success**
with an empty vector. Every consumer was told the account held nothing:
startup position seeding, the reconcile gate, the risk projection, the
dashboard's position panels.

The account held twelve positions, including a short in a long-only
strategy that the empty book had been concealing.

**Why it survived.** Both tests covering the decode encoded the same
phantom field. They agreed with the decoder and passed. A test written
from the same misreading as the code confirms the misreading.

**What changed.** The field is gone, the tests are rewritten against a
live wire capture, and a row that fails to decode no longer shrinks the
book — it marks the whole snapshot UNKNOWN, the same as a timeout, and
logs the raw frame. Two consumers were corrected alongside it: startup
seeding no longer converts a short into a long by taking `abs(qty)`, and
the dashboard's reconciler no longer treats an absent expectation as an
expectation of zero.

**What it should change in how you read this page.** Any claim on this
page about what the account held, dated before 2026-08-04, was reading
this sensor. Two entries above have been corrected in place; both
corrections are marked. Treat the remainder with the same suspicion —
a defect this far upstream does not announce which conclusions it
touched.

Both defects of this shape in the same file were **fixed in v0.4.0**. On
the open-orders path, a row that will not decode now poisons the whole
request — `list_open_orders()` returns `MalformedCsv` rather than a
shorter book — see the `open_orders_decode_failed_` latch in
`src/net/ibkr_connection.cpp` (cited by symbol; the line numbers this
entry used to carry have since moved) — so
"we could not determine what is open" is no longer indistinguishable
from "nothing is open" to a caller whose entire job is cancelling
whatever is open. On the account-value path the snapshot is no longer
sealed when a field failed to decode (`:1344-1368`, `:1416-1426`).

The position-data lesson above stands unchanged — it is about
conclusions already drawn from the broken sensor, which fixing the
sensor does not retract.

### 2026-08-01 — a host reboot, and the watchdog behaving correctly

Recorded because a control that works is also worth dating. A host
reboot took the daemon down mid-session; the watchdog escalated as
designed — soft pause at 15:51:24, kill on broker disconnect at
15:51:57.

### 2026-08-01 — a 0 % win rate published beside a +475 % return

**What happened.** Every portfolio run reported `"win_rate": 0.0` and
`"avg_win_loss": 0.0` in `results.json` and printed `Win rate 0%` on the
console — on runs with 938 to 1318 trades and returns above +400 %. A
research batch published tables carrying it.

**Root cause.** Not a miscalculation. The metric is deliberately not
computed on that path, because pairing round-trip trades across legs
does not decompose cleanly. That reasoning is fine. Emitting `0.0` for
it is not: the JSON schema already carried `null` elsewhere and the
console already rendered non-finite values as `—`, so "not computed" was
expressible, and this path chose the one value indistinguishable from a
real result.

**Why it is on this page.** A 0 % win rate beside a +475 % return is
absurd enough to catch by eye. The same pattern on a metric with a
plausible zero would not be, and there is no reason to think this was
the only instance of the pattern.

**What changed.** `null` in JSON, `—` on the console, and
`provenance.not_computed` names the metric and the reason. ~~One path
(multi-symbol walk-forward) still reports `0.0`, deliberately, because
that value feeds parameter selection — provenance says exactly that
rather than claiming a null.~~ **Superseded.** That path was
the interesting one, and deferring it left the worse half of the defect
live: the fabricated `0.0` was not just printed, it was *ranked*. The
optimizer's argmax compared it against real scores and, when the
unmeasured cell came first in enumeration order, it won — every
comparison against a NaN is false, so `max_element` never moved off it.
Absence is now representable end to end (`std::optional<double>`
through the comparator), a measured cell always outranks an unmeasured
one, and a window where nothing was rankable says so in
`best_score: null` plus a console error instead of presenting the first
enumerated cell as an optimum.

### 2026-08-01 — a drawdown comparison that was not like-for-like

**What happened.** A validation matrix reported a strategy's maximum
drawdown as −14.29 % against a buy-and-hold benchmark's −48.13 %, and
read that as the strategy's main advantage. A walk-forward pass then
found that `execution(compound = ...)` defaults to false and no config
in the study set it, so the strategy's risk per dollar of equity decayed
2.6× across the sample while the benchmark's did not.

**What it means.** The drawdown comparison was between a book running at
roughly 40 % of the benchmark's risk at the moment being compared. Under
walk-forward, where every segment restarts un-decayed, the same
strategy's pooled drawdown was −38.82 % against the benchmark's
−36.90 % — no advantage at all.

**What was and was not retracted.** No published number was restated;
they are all arithmetically correct measurements of what was run. The
*interpretation* was withdrawn. The return comparison survives — the
strategy did not beat buy-and-hold on return either way — and the
drawdown comparison does not.

**What changed.** The `compound` interaction is documented in the
[parity matrix](safety-and-limitations.md#compound-true-has-no-live-equivalent) above. A
like-for-like fully-invested comparison is still not expressible on this
build, because `compound = true` interacts with the gross-exposure
limitation and produced a −112 % drawdown in the same study. That is an
open gap, not a fixed one.

~~Documenting it was the whole remedy.~~ **Superseded.** The
sentence above described a finding that only a reader who already
knew to look for it would ever meet: the 2.6× was produced by a
walk-forward pass run *because someone was suspicious*, and nothing in
the engine would have volunteered it. The run now measures its
own sizing-basis range and ships a `sizing_basis_decay` warning in
`provenance.warnings` when it reaches 1.25×, so the artifact that
carries the drawdown also carries the reason not to compare it. The
underlying gap — no expressible constant-risk comparison — is still
open, and the default is still `false`; see [the section
below](safety-and-limitations.md#the-decay-is-now-measured-and-the-default-still-does-not-flip)
for why flipping it would have made things worse rather than better.

---
