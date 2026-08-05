# Daemon eval self-healing

The daemon's eval pipeline is the place a strategy bug can cause
real money damage silently: a Thursday-close eval throws, no orders
dispatch, no obvious alarm, and the operator only notices at Friday
open with 17.5 hours of unhedged basis already on the book.

EPIC-75 hardens that path with five composable capabilities, all
enabled by default, all driven by a single new canonical record —
the `EvalRound`.

| Capability             | Today (v1)                                            |
|------------------------|-------------------------------------------------------|
| **EvalRound logging**  | Every eval (ok / empty / error) writes one journal row |
| **Idempotency**        | Same close_ts re-eval drops or alerts; never double-orders |
| **Pre-trade replay**   | Re-runs yesterday's eval, alerts on divergence         |
| **Missed-eval detect** | Background sweep, alerts when expected close had no eval |
| **Backfill** (P4b)     | Replays a missed close eval and submits real orders — manual verb, CLI, or auto on broker reconnect |

## What gets recorded

Every evaluation the daemon runs — successful, empty, or thrown —
produces one `EvalRound` row in `events.jsonl`. The shape:

```json
{
  "kind": "eval_round",
  "ts_ns": 1733424000000000000,
  "payload": {
    "close_ts_ns": 1733424000000000000,
    "evaluated_at_ns": 1733424000732000000,
    "strategy_id": "deploy_it_long_only.qe",
    "cross_sectional": true,
    "status": "ok",
    "legs": [
      {"sym": "AAPL", "entry": 1.0, "exit": 0.0, "in_pos": false,
       "decision": "buy", "qty": 10.0},
      {"sym": "MSFT", "entry": 0.0, "exit": 0.0, "in_pos": false,
       "decision": "none", "qty": 0.0}
    ],
    "hash": 14883920138472958301
  }
}
```

`status` is one of:

| value     | what it means                                             |
|-----------|-----------------------------------------------------------|
| `ok`      | Eval produced ≥1 order. Normal path.                     |
| `empty`   | Eval ran but emitted **zero** orders. **The silent-fail shape EPIC-75 is built to catch.** |
| `error`   | Evaluator threw. `error_message` populated.              |

`hash` is the **idempotency hash** — a stable digest over the
discrete decision outcome (symbol, decision, qty rounded to 1e-6
shares, close_ts_ns, strategy_id). It deliberately ignores raw
signal scalars (which can drift in the last ULP) and observation
metadata (`evaluated_at_ns`). Two evals that produce the same
broker effect produce the same hash.

## Idempotency

The daemon maintains an in-memory `(strategy_id, close_ts_ns) →
EvalRound` index, hydrated from the journal at startup. Every
incoming round consults the index:

| Outcome                    | Daemon action                          |
|----------------------------|----------------------------------------|
| **Fresh**                  | Append `EvalRound` to journal           |
| **DuplicateEquivalent**    | Drop silently (debug-log only)         |
| **DuplicateViolation**     | Append `EvalIdempotencyViolation` event; **refuse** the new round |

`DuplicateEquivalent` is the common path for operator-triggered
re-eval. A re-run after a daemon restart, a manual replay, or a
pre-open verification all hit this path and **never double-order**.

`DuplicateViolation` is the dangerous case: the same close_ts now
produces different targets. The daemon records the violation but
preserves the original — the operator decides how to resolve.

## Pre-trade replay

Run on demand (P5 control verb is the v1 surface; auto-schedule at
T-10min before each session open lands in a follow-up):

1. Look up the journaled `EvalRound` for `(strategy_id, close_ts_ns)`.
2. Re-run the strategy in-process against the daemon's `BarHistory`
   (rolling cache of the last ≤500 closed bars per symbol).
3. Diff the new `idempotency_hash` against the journaled one.

Outcomes:

| outcome              | meaning                                        |
|----------------------|------------------------------------------------|
| `Verified`           | Hashes match — eval pipeline healthy           |
| `Mismatch`           | Hashes differ — `EvalReplayMismatch` journal event raised, F6 panel shows red badge |
| `NoJournaled`        | No round at close_ts — fall through to missed-eval detector |
| `InsufficientBars`   | Bar cache doesn't cover close_ts (e.g. fresh restart) |
| `Error`              | Engine construction failed                      |

Replay is read-only: a `CapturingNullRouter` redirects the fresh
engine's submit path before any bar is dispatched, so replay **can
never reach the broker** even by accident.

## Missed-eval detection

A 60s background sweep (`MissedEvalWorker`) asks the detector:

- What's the most-recent session close at least 60s in the past?
- Does the `EvalRoundIndex` have a row for that close?
- If not, journal `EvalMissed` (deduped per close so the same gap
  doesn't flood the journal).

The dedup set is rebuilt from the journal at startup, so a daemon
restart between alerts doesn't re-alert for the same gap.

The first sweep is delayed by one full interval so a cold-start
daemon doesn't false-alert before its first eval has a chance to
land.

## Backfill (same-evening recovery, P4b)

Detection alone leaves the operator holding the bag: the alert
fires, but the orders the close eval would have produced still
don't exist. `qe::live::EvalBackfiller` closes that loop. Given a
missed `close_ts_ns` it:

1. Refuses if `EvalRoundIndex` already has a round for
   `(strategy_id, close_ts_ns)` → `already_handled`.
2. Refuses if `now - close_ts_ns > max_age_seconds` (default
   **8 h**) → `out_of_window`. Same-evening only: for a daily
   `fill_model = "next_open"` strategy a 16:30 ET and a 22:00 ET
   backfill produce identical orders, but after the next session's
   open a backfill would chase yesterday's signal at today's
   prices.
3. Replays the eval against the daemon's `BarHistory` (the same
   `replay_eval_for` machinery pre-trade replay uses — historical
   bars can't fire orders during the replay's warm-up).
4. **Claims the close in `EvalRoundIndex` via `try_record` BEFORE
   submitting.** Only a `Fresh` outcome proceeds; everything else
   returns `already_handled` without touching the router. This is
   what makes repeated triggers — manual re-runs, TWS-1102
   reconnect storms — collapse to exactly one submission. (The
   2026-06-10 paper incident submitted the same round 17× during
   Gateway flapping because the index was never updated
   post-backfill.)
5. Translates the round's per-leg `decision = "buy" | "sell"` into
   `OrderIntent`s and submits through the daemon's **real** router
   — the same code path a live close uses.
6. Journals the round as `eval_backfilled` (not `eval_round`) so
   replay and the dashboard can tell a backfilled round from a
   live one. `EvalRoundIndex::populate_from_journal` indexes both
   kinds, so a daemon restart remembers prior backfills and the
   reconnect trigger doesn't re-submit them.

### Trigger paths

Three ways to fire a backfill, all routing through the same
`backfill_one` (so a double-trigger is a no-op):

- **Control verb** — `eval_backfill_now` with
  `{"close_ts_ns": <int>}` (explicit) or `{"latest_missed": true}`
  (scans the recent-events ring back-to-front for the newest
  `eval_missed`).
- **CLI** —

  ```bash
  qe_daemon backfill 1781035200000000000   # explicit close_ts_ns
  qe_daemon backfill --latest              # newest eval_missed
  ```

  Exit codes: `0` for `ok` / `already_handled` (no-op success),
  `3` for any refusal (`out_of_window`, `insufficient_bars`,
  `no_round`, ...), so shell pipelines can branch.
- **Auto-trigger on reconnect** — when IBKR raises TWS `1101` /
  `1102` ("connectivity restored"), the daemon checks whether the
  most recent past session close (with a 5-min post-close grace,
  so the catch-net doesn't race a live eval about to land) has a
  journaled round, and backfills it if not. `1100` (connectivity
  *lost*) never triggers — replaying mid-outage would race the
  recovery itself. This is the fix for the classic bug shape: a
  16:01 ET TWS bounce eats the close bar, the live eval never
  runs, and the daemon heals itself the moment connectivity
  returns.

Daemon warmup also seeds `BarHistory` with the warmup bars
(`engine.warmup()` alone primes only the indicators), so the
backfiller can replay the most-recent close even when the live
stream never received it.

### Statuses

| status                   | meaning                                          |
|--------------------------|--------------------------------------------------|
| `ok`                     | Round replayed, orders submitted (possibly 0 if every leg resolved `none` — the `eval_backfilled` event still records the empty round) |
| `already_handled`        | Index already has the round (live eval landed, or a prior/concurrent backfill won the claim) — no-op |
| `insufficient_bars`      | `BarHistory` doesn't cover `close_ts_ns`; v1 doesn't re-fetch history from IBKR |
| `out_of_window`          | Close older than `max_age_seconds` — session has rolled past |
| `engine_construct_failed`| Replay engine construction failed (config-level error; `error` populated) |
| `no_round`               | Replay ran but produced no round for that close (e.g. cross-sectional barrier unsatisfied with available bars) |

One sharp edge, by design: the index claim happens before the
router submit, so if every submit fails the close stays claimed
and a retry needs operator intervention (journal surgery). We
prefer that over the duplicate-submission failure mode.

### What backfill does NOT do

- **Staged entry / exit** — strategies with `entry_schedule` /
  `exit_schedule` bypass `OrderScheduler` slicing on the backfill
  path in v1 (whole-qty submit, logged when hit). Deliberate scope
  cut: P4b unblocks daily mean-rev / XS rebal first.
- **Multi-close recovery in one call** — the caller loops over
  `close_ts_ns` values.
- **History re-fetch** — `insufficient_bars` is terminal in v1.

## Dashboard surface

The F6 TRADE panel shows one EVAL line directly below the DAEMON
DETAILS line:

```
DAEMON · pid 12345 · up 1h 23m · 4520 bars · 12 orders · cfg foo.qe
EVAL   · last 2026-06-04 20:00Z ok 3 orders  ·  2 alerts
```

- Colours: green ok / yellow empty / red error or alerts.
- Hidden when daemon attaches but has no eval activity yet (cold
  start before the first close).

Cache poll cadence is the same EPIC-72 idle-backoff loop the other
daemon panels use: 3s when active, 60s when 5 consecutive polls
land identical orders / positions / equity / status / eval / log.

## Control verbs

| verb               | request                                  | response                          |
|--------------------|------------------------------------------|-----------------------------------|
| `eval_history`     | `{"tail": 50}` (optional, default 50)    | `{"rows": [EvalRound, ...]}` newest-first |
| `eval_alerts`      | `{"tail": 50}` (optional, default 50)    | `{"rows": [{kind, ts_ns, payload}, ...]}` newest-first |
| `eval_check_now`   | `{}`                                     | `{"ok": true}` or `InvalidConfig`  |
| `eval_backfill_now`| `{"close_ts_ns": <int>}` or `{"latest_missed": true}` | `{status, close_ts_ns, resolution, orders_submitted, round?, error?}` |

## Configuration

EPIC-75 ships with sane defaults; nothing is configurable from
`.qe` yet (deliberate — the v1 knobs are operational, not
strategy-level). The hard-coded values:

| knob                          | default | rationale                  |
|-------------------------------|---------|----------------------------|
| Idempotency dedup window      | ∞       | journal-scan based, no time-out |
| Bar history per-symbol cap    | 500     | 1 day of minute bars or 1 year of daily |
| Missed-eval sweep interval    | 60s     | balance alert latency vs CPU |
| Missed-eval post-close grace  | 60s     | avoids 16:00:01 false alarms |
| Backfill max age (`max_age_seconds`) | 8 h | close at 16:00 ET stays backfillable until next 09:30 ET open; daily-strategy assumption |
| Reconnect-backfill post-close grace | 5 min | don't race a live eval that's about to land |
| Pre-open replay lead (Phase 5b auto) | 10 min | first-fix window before market |

## Failure modes

| symptom                                | likely cause                                | what to check |
|----------------------------------------|---------------------------------------------|--------------|
| F6 shows `EVAL last ... empty (0 orders)` | Cross-sectional barrier didn't reach quorum; bars from one universe symbol stuck |
| Red `N alerts` chip with `eval_missed`  | Daemon was offline at session close, OR strategy threw silently. Same evening: `qe_daemon backfill --latest` recovers the round (a TWS-1101/1102 reconnect auto-attempts the same thing) |
| Red chip with `eval_replay_mismatch`    | Signal pipeline drift between live and replay — usually a bug |
| Red chip with `eval_idempotency_violation` | Two evals at the same close_ts produced different targets — operator action needed |

The right starting point for any of these is the journal:

```bash
tail -F ~/Library/Application\ Support/qe_daemon/state/<config>/events.jsonl |
    jq 'select(.kind | startswith("eval_"))'
```

## Out of scope

- **Backfill of staged-entry slicing** — backfilled rounds submit
  whole quantities; `OrderScheduler` slicing isn't wired through
  the backfill path yet (see "What backfill does NOT do" above).
- **Cross-daemon HA** — if two daemons run the same strategy,
  idempotency is per-daemon; they'd race. Out of scope.
- **External alerting (Slack / PagerDuty)** — v1 ships journal +
  spdlog. Webhook integration is a follow-up.
- **Replay diff visualization** — beyond match/mismatch. A
  side-by-side targets diff lands when the EVAL HISTORY collapsible
  panel ships.
