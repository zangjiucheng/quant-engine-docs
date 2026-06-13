# Live-trading safety model

What stops a bug or a wrong keystroke from blowing up your
brokerage account when `qe_dashboard` starts submitting orders
for real.

This page is the **threat model + design rationale** behind the
five-layer safety net. The implementation lives in
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
   without crashing the dashboard or sending stale orders on
   reconnect.

## Five layers of defense

Each layer is independent. Bypassing one still leaves four. The
order below is roughly "outermost (config-time) → innermost (per-
order check)" — a bad order has to pass all of them to reach the
broker.

### Layer 0 — config opt-in

**What it stops:** mistaken-account (class 1).

Live mode requires **both** of:
- env var `QE_LIVE_TRADING=I_MEAN_IT` set in the shell that
  launches `qe_dashboard`. Missing → live config is rejected at
  load time, dashboard refuses to start. Cannot be set via
  dashboard UI, cannot be persisted via config file. Has to be
  re-affirmed every session.
- Settings (Cmd+,) broker mode set to `ibkr-live` (default
  `alpaca-paper`).

Both must be true to reach Layer 1. Paper modes (alpaca-paper,
ibkr-paper) skip the env-var requirement so iteration is fast.

### Layer 1 — startup confirmation modal

**What it stops:** mistaken-account (class 1, after Layer 0
opt-in was deliberate).

Live mode boot blocks on a modal that:
- Shows account number (e.g. `U1234567`) fetched from broker.
- Shows current NetLiq (e.g. `CAD $42,310.50`).
- Requires user to **type the full account number into a text
  field** to dismiss. Approximate-match is not accepted (no auto-
  complete, no first-N-chars).
- Has no "remember me" button. Re-prompts every session.

Modal cannot be background-dismissed (no click-outside, no Esc).
Cancel button forces broker disconnect + return to PAPER mode.

### Layer 2 — per-order limits (`SafeBroker`)

**What it stops:** mistaken-quantity (class 2), runaway loop
(class 3).

`SafeBroker` wraps any concrete `IBroker` and rejects orders
before they hit the wire when any of these are violated:

| Limit | Default | Source |
|---|---|---|
| `max_notional_per_order` | `CAD $5_000` (live) / unbounded (paper) | Settings (Cmd+,), config-persisted |
| `max_position_per_symbol` | 200 shares (live) | Settings (Cmd+,) |
| `max_orders_per_minute` | 10 (live) / 60 (paper) | hard-coded, Settings (Cmd+,) read-only |
| `max_daily_loss` | -CAD $500 (live) | Settings (Cmd+,); triggers kill-switch |
| Allowed symbols allow-list | empty = all (paper) / explicit list (live recommended) | Settings (Cmd+,) |

Rejections log to `orders.jsonl` with reason + intended order
payload + timestamp. Strategy gets a clear error, doesn't crash.

Limits are **multiplicative with broker-side limits**, not a
replacement. IBKR has its own server-side checks (margin,
short-locate, restricted symbols); SafeBroker stops the order
before the wire so the request never reaches them. Belt + braces.

### Layer 3 — kill-switch

**What it stops:** catastrophic strategy failure (class 4).

`Cmd+Shift+K` from any screen:
1. Sets `cancel_all_open_orders` on the broker (best-effort —
   logs any failures).
2. Disables every running strategy (sets `running_ = false`; new
   bars no longer trigger `on_bar`).
3. Logs a `KillSwitch` event to `orders.jsonl` with timestamp.
4. Shows a status modal: `Killed at HH:MM:SS · cancelled N
   orders · CAD $X at risk in open positions.`

**Kill-switch does NOT auto-flat positions.** Closing a position
is a decision (could be lock-in of profit or realization of a
big loss); it's a human call, not a panic-button default. The
modal makes the open notional explicit so the user can choose.

Kill-switch is always-on — works in paper modes too so the
muscle memory is there before live trading.

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

The same loop pulls `accountSummary("DayTradesRemaining")` on
IBKR and pops a warning when it hits 1 (so the user knows the
next round-trip will use a PDT day; helpful even for IBC accounts
since US securities can still trigger PDT-like restrictions).

### Layer 5 — gateway health (IBKR only)

**What it stops:** broker / network failure (class 6).

TWS socket disconnect:
- Strategy auto-disables (same effect as kill-switch, minus the
  cancel — we can't reach the broker to cancel anyway).
- Top-bar badge flips to `BROKER OFFLINE` (red, blinking).
- Polling backs off (don't hammer a dead socket).

Reconnect (manual via Settings (Cmd+,) → Retry, never auto):
- Strategy stays disabled until human re-enables.
- Modal: `Reconnected. Reconcile? Y/N` — runs the Layer 4 loop
  once before enabling order submission.

We **never auto-restart TWS**. The TWS login flow has 2FA + a
race-with-user dialog; restarting is the IBKR side's job.

## Phase delivery

The layers don't have to land together. Phase A (the first commit
stack on this branch) ships layers 0 / 2 / 3 / 4 with Alpaca
paper as the test target. Phase B adds layer 5 (IBKR-specific)
when the TWS adapter lands. Phase C adds the UX (kill-switch
chord, position-drift modal, badge). Layer 1's startup modal
only fires in `ibkr-live` mode and lands in Phase B/C.

That means the branch is shippable in stages:
- After Phase A: hardened paper trading on Alpaca, with kill-
  switch + safety layer + reconcile. Usable for dev work.
- After Phase B: ibkr-paper works end to end with the same
  safety layer.
- After Phase D: ibkr-live works with full layer 1 + layer 5.

## What we deliberately do NOT do

These would be reasonable safety measures we choose not to
implement, with the reasoning:

| Skipped | Why |
|---|---|
| Auto-flat all positions on kill-switch | Closing a position is a P&L decision. Wrong default. |
| Auto-reconcile drift | Silent state correction is how money disappears unnoticed. |
| Auto-restart TWS | Races with the 2FA login prompt; out of our process scope. |
| Auto-resume after disconnect | A disconnect window is itself a state-uncertain period; reconnect should be an inspected event, not background. |
| Per-symbol stop-loss enforced by broker | Would need server-side STP orders pre-placed; v2 question. Layer 2's `max_daily_loss` is the v1 backstop. |
| `dry_run` mode that logs orders but doesn't submit | Paper modes (alpaca-paper, ibkr-paper) already cover this. A "live but not really" mode is exactly the configuration that creates Layer 0's class-1 incidents. |
| Two-person sign-off (multi-user approval) | Single-user project. Different scope. |

## See also

- [`docs/ibkr-connectivity.md`](ibkr-connectivity.md) — how to
  set up TWS or IB Gateway for the dashboard to talk to. Read
  before flipping `broker = "ibkr-paper"`.
- [`docs/ibkr-paper-verification.md`](ibkr-paper-verification.md)
  — the end-to-end paper smoke test you should clear before going
  live.
