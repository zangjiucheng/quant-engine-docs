# Live-trading safety model

What stops a bug or a wrong keystroke from blowing up your
brokerage account when `qe_daemon` starts submitting orders for
real.

!!! note "There is no order ticket any more"

    This page described two submitters: the daemon, and the C++
    dashboard's **manual trading** mode, which could place an order
    from the GUI. That app was retired in EPIC-96 and its successor,
    QE Fleet, deliberately has no order ticket and no plan for one —
    the only write it can send a venue is a cancel. Every layer below
    is now the daemon's. Where a layer existed only on the dashboard
    side, it says so.

This page is the **threat model + design rationale** behind the
layered safety model. The implementation lives in
`qe::broker::SafeBroker` + `qe::broker::IbkrBroker`; both are
tested against `AlpacaPaperBroker` as the zero-cost development
target before any real TWS connects.

Read this before flipping the env gate to live.

## Threat model — what can actually go wrong

There are six classes of failures we design against. Each is
matched by one or more layers below.

1. **Mistaken-account incident.** User intends to submit a paper
   trade, broker mode silently flipped to live (config drift, UI
   bug, env var leak across shells). Order lands on real account.
   Worst case: full position size on the wrong account.

2. **Mistaken-quantity incident.** Strategy emits a buy intent
   with wrong `size` (decimal-point typo, sweep override
   overflow, signal-layer arithmetic edge case). One order with
   100× the intended notional.

3. **Runaway loop.** Strategy or polling code submits orders in a
   tight loop because of a state-machine bug (e.g. failed cancel
   re-fires entry every bar). Rate-limit overrun, broker may
   throttle or close the session.

4. **Catastrophic strategy failure.** Strategy works as designed
   but the design itself loses money fast (regime change,
   unhandled gap, missing stop). No bug — just bad logic — but we
   need to stop bleeding once it's recognized.

5. **State desync.** Local view of positions / cash drifts away
   from the broker's view (missed fill notification, partial fill
   misaccounted, app crashed mid-order). Strategy makes decisions
   from a fictitious book.

6. **Broker / network failure.** TWS crashes, socket disconnects,
   broker rejects via session timeout. Need to stop trading
   without crashing the daemon or sending stale orders on
   reconnect.

## Layers of defense {#five-layers-of-defense}

Each layer is independent. Bypassing one still leaves four. The
order below is roughly "outermost (config-time) → innermost (per-
order check)" — a bad order has to pass all of them to reach the
broker.

### Layer 0 — config opt-in

**What it stops:** mistaken-account (class 1).

Live mode requires **both** of:
- env var `IBKR_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING` set in the
  environment that launches `qe_daemon`
  (`src/net/ibkr_broker.cpp`, token at
  `include/qe/net/broker.hpp`). Missing → the live broker
  refuses to construct. Cannot be set from any UI, cannot be
  persisted via config file. Has to be re-affirmed every session.

  Note what this does **not** do: absence of the variable is not a
  refusal to run — it is a refusal to build a real-money broker. A
  process that came up is not evidence the gate is open.
- `broker = "ibkr-live"` in the deployed `live(...)` config.

Both must be true to reach Layer 1. Paper modes (alpaca-paper,
ibkr-paper) skip the env-var requirement so iteration is fast.

### Layer 1 — startup confirmation modal

**What it stops:** mistaken-account (class 1, after Layer 0
opt-in was deliberate).

**This layer was the dashboard's and went with it.** It was a modal
that showed the account number and NetLiq and required you to type
the full account number to dismiss, every session, with no "remember
me".

What replaced it is not a dialog, because `qe_daemon` is headless and
deployed over SSH: the account is **asserted in the config and
checked against the broker**. `live(...)` names an
`expected_account`, and the daemon refuses to start if the account it
connected to is not that one. A four-way agreement — the manifest's
`broker`, the config's `broker`, the account-number prefix (`DU…` for
paper, `U…` for real money) and `expected_account` — is checked by
`scripts/qe-strategy-fleet.py` for every observed book, so a
mislabelled deployment is a reported disagreement rather than a
surprise.

That is a stronger gate than the modal in one way and a weaker one in
another: it cannot be clicked through, and it cannot ask a human
whether they meant it.

### Layer 2 — per-order limits (`SafeBroker`)

**What it stops:** mistaken-quantity (class 2), runaway loop
(class 3).

`SafeBroker` wraps any concrete `IBroker` and rejects orders
before they hit the wire when any of these are violated:

| Limit | Default | Source |
|---|---|---|
| `max_notional_per_order` | `0` — disabled | no config path forwards it |
| `max_position_per_symbol` | `0` — disabled | no config path forwards it |
| `max_orders_per_minute` | `0` — disabled | no config path forwards it |
| `max_daily_loss` | `0` — disabled. **Positive magnitude**, `<= 0` disables | the only one `qe_daemon` forwards (`apps/daemon/main.cpp`) |
| Allowed symbols allow-list | empty = all | no config path forwards it |

> **Layer 2 is the kill-switch short-circuit only — no per-order cap
> is armed.** `SafeBrokerLimits` default-constructs every field to
> `0`, which means disabled (`include/qe/net/safe_broker.hpp`), and
> the daemon forwards exactly one of them.
>
> Arming the rest would mean a new `live(...)` keyword and a new
> forwarding site; today only `max_daily_loss` reaches this layer.
> Treat Layer 2 as a backstop that is *available*, not as one that is
> *on*.

Rejections log to `orders.jsonl` with reason + intended order
payload + timestamp. Strategy gets a clear error, doesn't crash.

> `SafeBrokerLimits` is the submit-path wrapper; it used to be
> described here as "the dashboard-side layer", which stopped being
> true when the dashboard was retired — `qe_daemon` is the only thing
> that constructs one now (`apps/daemon/daemon_wiring.hpp`,
> `apps/daemon/main.cpp`). The daemon's *pre-trade* gate,
> `qe::live::RiskLimits` (configured from `.qe`
> `live(risk_max_* = ...)`), is a different object with the same
> convention since EPIC-83: **every limit is a positive magnitude
> and `<= 0` disables it**, and `risk_max_position_per_symbol` there
> is USD notional rather than shares.
>
> The two daily-loss checks measure **the same thing** — and this page
> said otherwise until 2026-08-11. Since EPIC-88 T88.5 this one no
> longer reads the broker's `day_pl`: it calls the injected
> `strategy_daily_pnl` callback (`src/net/safe_broker.cpp:220-221`),
> so both are the *strategy book* against the same session anchor,
> not the account. They differ only in what they do about it — this
> one **rejects the order in hand** (`src/net/safe_broker.cpp:216-245`;
> it never calls `trip()`), while the pre-trade gate trips the
> daemon's KillSwitch (`src/live/pre_trade_risk.cpp:525-547`). Feed
> them the same number.
>
> **With no `strategy_daily_pnl` source wired, this layer measures
> nothing** — it is inert rather than permissive-by-accident
> (`include/qe/net/safe_broker.hpp`), but inert is still not
> protection. Check that the deployment wires one before counting it
> as a layer.
>
> **Both are evaluated at submit time, not continuously.** Neither
> is a monitor watching equity in the background: each one tests its
> threshold inside the submit path, so a breach that happens while
> nothing is being submitted sits latent until the next order
> attempt — and is then judged on the equity current at *that*
> moment. On a `rebalance = N` deploy that gap is roughly N
> sessions. What *is* continuous is reporting: `qe_daemon status`
> re-evaluates `daily_loss_breached` on every poll with no side
> effects, so the latent breach is visible even though nothing has
> fired.
>
> **The notional caps are evaluated against a *projected* book, and
> that projection is reconciled** (EPIC-83 T83.23). The daemon's
> gate counts an order against your position the moment the router
> accepts it — not when it fills — so the caps still bind partway
> through a ten-leg rotation instead of measuring every leg against
> the pre-batch book. The unconfirmed half of that projection is
> given back when the venue rejects the order or its reservation
> expires unfilled, and the whole projection is re-anchored to
> `list_positions()` once a minute while the market is open. If the
> position query fails, the re-anchor is skipped — an unanswered
> query is never treated as a flat account. `qe_daemon status`
> reports `projection_releases`, `projection_reseeds` and
> `projection_corrections`; corrections climbing while releases stay
> at zero means an order is being booked and never given back.
>
> **The daemon's gate also refuses a SELL that would leave the account
> short**, and this one is not configurable — it is not a `risk_max_*`
> knob and there is no way to turn it off from a `.qe`. `LiveEngine`
> enters with BUY, exits with SELL and cannot represent a short
> position at all, so an exit sized larger than the account holds is a
> defect in every config rather than a preference in some. Two details
> worth knowing when you read a rejection: it measures the **account's**
> projected book, not the strategy-scoped one the exposure caps use, so
> a strategy exit against shares you also hold by hand does not trip
> *this* check — though once you declare a baseline it may trip the
> strategy-scoped one described immediately below, which is a separate
> gate with a separate reason string; and it is
> evaluated outside the risk-reducing exemption those caps share, since
> a sell of 8 against a long 5 lands at −3 and is "smaller" by
> magnitude than the position it just inverted. The log line and the
> journal's `order_rejected` row read `long-only violation`; the status
> is `risk_rejected`, like every other refusal from this gate, and like
> every other one it is not retried.
>
> **The same flag arms a second refusal, on the other book**.
> Once you declare a baseline, the account and the strategy are no
> longer the same set of shares, and a sell can leave the *strategy*
> short while the account is still comfortably long — at which point
> none of the checks above sees anything wrong: the risk-reducing
> exemption is account-scoped and skips both caps, and the direction
> check above reads the account and finds it long. Measured on the real
> gate at 100 shares held, all 100 of them declared as yours, a sell of
> 40 was accepted and left the strategy $6,385 short of stock only your
> baseline could back. So a SELL that would leave the strategy short
> **while the account stays long** is refused, reason `strategy short
> inversion`. It cannot fire without a declared baseline, and it
> deliberately does not fire when the sell takes the account to flat —
> that is the ordinary "exit a holding you also bought by hand" case,
> and blocking it would halt the whole rotation. The usual cause is a
> baseline that has gone stale because you hand-sold; the fix is to
> re-adopt it, not to change a limit.

Limits are **multiplicative with broker-side limits**, not a
replacement. IBKR has its own server-side checks (margin,
short-locate, restricted symbols); SafeBroker stops the order
before the wire so the request never reaches them. Belt + braces.

### Layer 3 — kill-switch

**What it stops:** catastrophic strategy failure (class 4) —
specifically, the *next* order. Not the last one.

> **Read this sentence before you rely on anything else on this
> page.** A kill-switch in this codebase **blocks new
> submissions**. It does **not** cancel an order that is already
> working at the venue, and it does not flatten a position. No
> keystroke and no button pulls a working order out of IBKR; the
> only things that do are the explicit `qe_daemon cancel-all`
> verb (EPIC-88 T88.1) and TWS / IB Gateway / Client Portal by
> hand. Both are separate, deliberate acts — see
> [Emergency stop](#emergency-stop-the-three-steps) below.

#### What tripping it actually does

There is **one KillSwitch, in the daemon process**, and it controls
only that process's submissions:

| | Daemon (`qe_daemon`) |
|---|---|
| Constructed at | `apps/daemon/main.cpp` |
| Tripped by | `DisconnectWatchdog`; continuous daily-loss monitoring and the submit-time backstop; `qe_daemon kill`; the `kill` control verb, which QE Fleet's F6 sends after two deliberate clicks |
| Effect | closes `OrderScheduler`'s submit gate → pending, not-yet-submitted slices are **preserved**, not cancelled: the killed branch dispatches nothing and destroys nothing (`src/live/order_scheduler.cpp`), and a restart re-evaluates them; writes one `Notice` row to the state journal; **raises a desktop notification** (EPIC-88 T88.8 → `src/live/operator_alert.cpp`) — journal first, toast second, and the toast can neither delay nor fail the trip |
| Does NOT | cancel at the venue; exit the process; stop fills on already-working orders from being booked (the fill path never reads kill state) |

This page used to describe a second switch, in the dashboard, guarding
that app's own manual submissions. Both the switch and the submissions
are gone. What survives is the distinction that mattered: a client's
button is not the daemon's switch. QE Fleet's F6 does not trip a local
anything — it sends the `kill` verb to the daemon, and only on a book
whose manifest names a control socket. **Cancel daemon venue orders**
is a separate action and reports partial failures and unknown
outcomes. Killing the daemon cancels no venue orders.

**Strategies keep running.** `LiveEngine` holds no reference to
any `KillSwitch` — bars keep arriving, `on_bar` keeps firing,
signals keep being produced. They are stopped one layer further
down, at the submit gate. The switch is a gate, not an off
button.

**The daemon does not exit on trip.** It keeps running, keeps
consuming market data, and keeps booking fills for orders that
were already working. Stopping the process is a separate,
deliberate step — and it must come *second*, see below.

#### What it deliberately does NOT do

**No auto-flat.** Closing a position is a decision (which leg
first, market vs limit, how much); it's a human call, not a
panic-button default.

**No auto-cancel.** A trip gates new submissions and touches
nothing already at the venue. This is operator decision D1
(EPIC-88, 2026-07-30), not an omission: cancelling is a decision
with its own confirmation, and folding it into a trip means every
watchdog flap starts pulling live orders.

`qe::net::cancel_all_open` (`src/net/kill_switch.cpp`) is how you
cancel, and since EPIC-88 T88.1 it has exactly one production
caller — the daemon's `cancel_all` control verb, i.e. somebody
typing `qe_daemon cancel-all` and confirming it. It is **not**
registered as a `KillSwitch` trip observer and must not become
one; `tests/test_daemon_control_handler.cpp`
(`"cancel_all is NOT reachable from a kill-switch trip"`) trips
the switch through every site and asserts the broker saw zero
cancels.

Until EPIC-88 T88.2 there was one code path that *attempted* a
venue cancel — `cancel_open_on_trip`, wired to the
`DisconnectWatchdog` on-trip callback rather than to a KillSwitch
observer. It was dead twice over: only an `ibkr_disconnect` trip
could reach it, and by then the link a cancel needs is the link
that just died, so `IbkrConnection::list_open_orders` answered
"not connected" (`src/net/ibkr_connection.cpp:307-308`) and it
returned early, every single time. T88.2 deleted it rather than
repairing it, because under D1 it should not exist at all. What
the watchdog writes now is the honest count of orders left
working — or the word "unknown" when it cannot ask.

#### Emergency stop — the three steps

If you need trading to stop *now*, with orders working:

```bash
qe_daemon kill --yes         # 1. nothing new goes out (one-way)
qe_daemon cancel-all --yes   # 2. clear what is already working
qe_daemon stop               # 3. only once step 2 came back clean
```

Step 2 is the only thing in this repo that pulls a working order;
the other way is TWS / IB Gateway / Client Portal, by hand. **Read
its report** — exit 0 means the venue listed the book and
acknowledged every cancel in it, exit 3 means the book may not be
clear, and a link that could not be listed prints "UNKNOWN"
rather than a count. See the runbook's
[Emergency stop](live-trading-runbook.md#emergency-stop-orders-are-working-and-i-need-them-gone)
for the exact output of each case.

**The order matters.** Between stopping the daemon and cancelling
at the venue, a working order can still fill — the venue does not
care that the process that sent it is gone, and with the daemon
dead nothing is left to book, journal, or react to that fill. Stop
the process last.

Step 1 is not optional bookkeeping either: `cancel-all` does not
trip anything, so on a live daemon it clears the book and the
strategy refills it at the next evaluation. The report warns when
the switch is not tripped.

Kill-switch is always-on — works in paper modes too so the
muscle memory is there before live trading. It is also **one-way**
within a process: once tripped it stays tripped for the process
lifetime, and clearing it means a restart.

#### Provenance of this section

Rewritten 2026-07-30 (EPIC-87) and **checked line-by-line against
the code at commit `c5c05a9`**; every file:line above was read,
not inferred. The previous version of this section was written
2026-05-29 in commit `45fb415` (T42.1) as *design intent*, before
the implementation existed, and was never reconciled with what
shipped. All four of its numbered claims turned out to be false.

Amended 2026-07-30 by EPIC-88, which changed the behaviour this
section describes rather than the description: T88.2 deleted the
dead `cancel_open_on_trip` callback, T88.8 added the trip
notification, T88.10 gave a client a way to kill the daemon,
and T88.1 / T88.11 added `qe_daemon cancel-all` and the
`kill` / `pause` / `resume` subcommands. What did **not** change is
the sentence at the top: a trip still blocks new submissions and
cancels nothing. That is decision D1, and it is now enforced by a
test rather than by convention.

The original wording also survives in
`EPICS/done/EPIC-42-live-broker-trading.md:282-284` (T42.11). That
file is a **historical record of what was planned**, so its task
text is deliberately left unedited — EPIC-87 only added a banner at
the top of it pointing here. Read it as intent, not as
documentation of behaviour, and prefer this page.

### Layer 4 — reconcile loop

**What it stops:** state desync (class 5).

Every 30 seconds (and on every startup):
1. `broker->get_positions()` → broker-side view.
2. `PositionsCache->snapshot()` → local view.
3. Per-symbol diff with tolerance (default: 1e-6 quantity).
4. Any diff above tolerance → modal: `Position drift detected.
   Adopt broker / adopt local / abort.` Three buttons; no
   default; modal blocks the UI.

We **never auto-reconcile**. This is brokerage state, not k8s
spec — silent adoption is the wrong default. Human decides.

In `qe_daemon` (which has no UI to put a modal in) the equivalent
since EPIC-89 T89.10 is a **submission gate**: drift raises a block,
every new order is refused with `reconcile_blocked`, and only
`qe_daemon reconcile-ack <id> --decision=<d>` clears it. The block is
journaled and rebuilt on the next start, so restarting is not a way
past it — a gate an operator can clear by restarting is not a gate.
The daemon still adopts nothing; the ack records the decision. Until
T89.10 the daemon logged the drift at WARN and kept trading, which on
2026-07-31 meant it went on submitting against ten fills its own
reconcilers had already contradicted.

> **Not implemented.** This paragraph described a
> `accountSummary("DayTradesRemaining")` poll that does not exist. The
> string has zero occurrences in any `.cpp` or `.hpp`, and
> `accountSummary` itself is not implemented at all — the IBKR link
> runs `reqAccountUpdates` exclusively, so the request is never sent
> and the reply could not be decoded if it were. **Nothing here
> watches the day-trade counter.**
>
> It does not bind on the deployed book, which rebalances every 20
> closes and has never day-traded, and the account is paper. It would
> bind on a live version: the book carries a 4.4% cushion over the
> $25,000 PDT line against a strategy whose measured max drawdown is
> -1.22%.
>
> Two things must be discovered before anyone builds it, in this order.
> First, whether IBKR delivers `DayTradesRemaining` on the
> `reqAccountUpdates` channel this daemon already runs — if it does,
> the work is one key handler; if not, it needs a new request, a new
> decoder and a new subscription lifecycle. Second, whether the paper
> account reports the value at all. Treating paper's silence as
> evidence the rule does not apply would be the same mistake as
> `trading_blocked`: a field that reads all-clear over a
> channel nobody is measuring.
>
> Whatever is built ships with a test that drives the warning, or it
> does not ship. And because "never received" and "received, and fine"
> are the same absence here, the operator surface has to distinguish
> them rather than render an unreported counter as a comfortable
> number.

### Layer 5 — gateway health (IBKR only)

**What it stops:** broker / network failure (class 6).

For `qe_daemon`, the disconnect watchdog has three states:

| State | Effect | Recovery |
|---|---|---|
| Healthy | Watchdog permits submissions; other risk gates still apply. | — |
| Paused | Short disconnect blocks new submissions; working venue orders remain. | Reconnection automatically clears the watchdog pause. |
| Tripped | Sustained disconnect trips the one-way kill-switch. | Inspect broker state and restart the daemon. Reconnection alone does not clear the kill. |

A configured maintenance window defers the hard trip while retaining the
pause and miss count. If the connection is still down after the window,
the next poll can trip. Pending staged slices survive the kill and restart;
expired windows become `window_miss`. No kill-switch cancels venue orders.

See the [watchdog state machine](architecture.md#disconnect-watchdog-state-machine-epic-70)
and [disconnect smoke checks](qe-daemon-smoke.md#verifying-the-disconnect-watchdog).
TWS/Gateway login and restart remain outside QE's control.

## What we deliberately do NOT do

These would be reasonable safety measures we choose not to
implement, with the reasoning:

| Skipped | Why |
|---|---|
| Auto-flat all positions on kill-switch | Closing a position is a P&L decision. Wrong default. |
| Auto-reconcile drift | Silent state correction is how money disappears unnoticed. |
| Auto-restart TWS | Races with the 2FA login prompt; out of our process scope. |
| Auto-resume after a hard watchdog trip | The kill-switch is one-way. A short disconnect only soft-pauses and can recover automatically. |
| Per-symbol stop-loss enforced by broker | Would need server-side STP orders pre-placed; v2 question. Layer 2's `max_daily_loss` is the v1 backstop. |
| `dry_run` mode that logs orders but doesn't submit | Paper modes (alpaca-paper, ibkr-paper) already cover this. A "live but not really" mode is exactly the configuration that creates Layer 0's class-1 incidents. |
| Two-person sign-off (multi-user approval) | Single-user project. Different scope. |

## See also

- [`docs/ibkr-connectivity.md`](ibkr-connectivity.md) — how to
  set up TWS or IB Gateway for the daemon to talk to. Read
  before setting `broker = "ibkr-paper"`.
- [`docs/ibkr-paper-verification.md`](ibkr-paper-verification.md)
  — the end-to-end paper smoke test you should clear before going
  live.
