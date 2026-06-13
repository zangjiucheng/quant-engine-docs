# IBKR connectivity setup

How to get TWS or IB Gateway running so `qe_dashboard` can talk
to Interactive Brokers. This page only covers the connectivity
prerequisites — the safety design behind what we do with the
connection lives in
[`docs/live-trading-safety.md`](live-trading-safety.md).

## TL;DR

Pick one once and stick with it:

| Choice | Use this when |
|---|---|
| **TWS Desktop** | You already use TWS for chart-reading or manual trading. Full UI; ~600MB RAM. |
| **IB Gateway** | You only want the dashboard's API access, no IBKR UI. ~150MB RAM, no charts. Recommended for `qe_dashboard`. |

Both speak the same protocol on the same socket. The dashboard
doesn't care which is on the other end.

Port map:

| Account | TWS port | IB Gateway port |
|---|---|---|
| Paper | 7497 | 4002 |
| Live  | 7496 | 4001 |

The dashboard's `broker = "ibkr-paper"` / `"ibkr-live"` setting
maps to the gateway port at connect time; the user picks which
host process is running.

## What to install

### Option A — IB Gateway (recommended)

1. Download from
   [interactivebrokers.com/en/trading/ibgateway-stable.php](https://www.interactivebrokers.com/en/trading/ibgateway-stable.php).
   The "stable" build is fine; "latest" is the newer release
   train. Either works.
2. Install. On macOS the installer drops a `.app` bundle and a
   `Jts` folder under `~/`.
3. Launch. The login window opens. Enter username + password +
   the appropriate 2FA method (IBKey, mobile push, or paper
   token depending on your account).
4. After login, the small Gateway window stays up. No charts,
   no order entry — just a status indicator showing the API is
   listening.

### Option B — TWS Desktop

1. Download from
   [interactivebrokers.com/en/trading/tws.php](https://www.interactivebrokers.com/en/trading/tws.php).
2. Install + launch + log in.
3. **Enable the API**: `File → Global Configuration → API →
   Settings`:
   - Check `Enable ActiveX and Socket Clients`.
   - Set `Socket port` to **7497** (paper) or **7496** (live).
     Confirm it matches what you'll set in `qe_dashboard` F8
     SETTINGS.
   - Uncheck `Read-Only API` (we need to submit orders).
   - In `Trusted IPs`, add `127.0.0.1` (the dashboard runs on
     the same machine).
   - Optional but recommended: leave `Master API client ID`
     blank so we can pick our own client id at connect time.
4. Click `OK`. TWS prompts a confirmation; accept it.

## Paper vs live

Same TWS / Gateway binary, different login.

- **Paper account login** uses the same username with a `paper`
  prefix (e.g. `myuser` → `myuserpaper`) or a separate
  paper-only username depending on when your IBKR account was
  set up. Check
  [IBKR's instructions](https://www.interactivebrokers.com/en/trading/free-demo.php).
- **Live login** uses your real account username.

You can only be logged into one at a time per running process.
To switch you log out + log back in with the other credentials.
That's deliberate on IBKR's side — there's no "swap account"
shortcut.

The dashboard's `broker = "ibkr-paper"` opens a socket on port
7497 / 4002 (paper). `broker = "ibkr-live"` opens 7496 / 4001
(live). If TWS is logged into paper but the dashboard tries
live (or vice versa), the connect fails with a clear error
because the wrong port isn't listening.

## Heartbeat + reconnect expectations

TWS / Gateway will:
- Auto-restart **once per day** on a configurable schedule (set
  in Settings → Lock and Exit). The 15-minute pre-restart
  warning fires; the dashboard's `BROKER OFFLINE` badge will
  flip during the restart window. Time the restart for after
  market close.
- Drop the socket on session timeout if no traffic for >30s.
  Our adapter sends `reqCurrentTime` every 20s as a keepalive.
- Reject API calls during the 2FA prompt window after login.
  Don't try to script around it — wait for the gateway to
  settle.

### What the daemon does on each connectivity event (EPIC-80)

TWS emits an `ErrMsg` system frame (`request_id = -1`) for every
Gateway-to-IB connectivity transition. The daemon handles each:

| Code | TWS message | `qe_daemon` action |
|---|---|---|
| `1100` | `Connectivity between IB and TWS has been lost` | Log a warn; wait for the matching restore frame. The socket itself is still up — no resubscribe is possible while data is dark. |
| `1101` | `Connectivity has been restored - data lost` | Re-arm every live market-data subscription: best-effort cancel of the stale `ticker_id`, then re-issue `subscribe_market_data` for each previously active symbol; atomically swap the internal ticker-id maps. |
| `1102` | `Connectivity has been restored - data maintained` | Same as `1101`. IBKR claims `1102` preserves subscriptions, but on IBKR Gateway 10.x we have observed sub-second flaps that leave the data farm dark until the next session even though `1102` was raised. Re-subscribing on `1102` is cheap (one socket write per symbol) and idempotent on the broker side; the wasted-work cost is negligible next to the cost of a silently missed close eval. |

Every connectivity event is recorded as a `notice` row in
`events.jsonl` (the `MissedEvalWorker` already journals the
downstream `missed_eval` alert if a close-eval doesn't land within
60 s — together those two rows pinpoint a flap-induced miss).

On top of the resubscribe, `1101` / `1102` also trigger the
**eval-backfill check** (EPIC-75 P4b): if the most recent past
session close has no journaled `EvalRound`, the daemon replays
that eval and submits the recovered orders — so the classic
"16:01 ET Gateway bounce eats the close bar" outage heals itself
the moment connectivity returns. `1100` never triggers a backfill
(connectivity just *died*; replaying then would race the
recovery). The backfill is idempotent — repeated `1102` frames in
a reconnect storm collapse to one submission. Details in
[Eval self-healing](eval-self-heal.md).

## Multiple dashboards / multiple client IDs

TWS / Gateway accept multiple simultaneous API connections,
each with a unique **client ID** (integer). The dashboard picks
`client_id = 1` by default. If you also use TWS API from a
notebook or another tool at the same time, set that other tool
to a different id (2, 3, ...) to avoid stepping on each other's
order acks.

## Account limits to know

These come up enough that they're worth flagging here even
though they're IBKR-side, not dashboard-side:

| Limit | Default | Notes |
|---|---|---|
| Market data lines | 100 (non-pro retail) | Watching too many symbols → "max market data lines exceeded" rejection. Unsubscribe what you don't need. |
| Messages per second | ~50 / client id | Our rate-limit (Layer 2) sits at 10 orders/minute on live — well under the broker limit. |
| Concurrent open orders | per-account, generous (>1000) | Practically not a constraint. |
| Order rejection rate | server-side throttle if too many rejects | Strategies that submit and get NACKed in a loop can trigger this. |

## Verifying the connection

Before flipping `broker = "ibkr-paper"` in Settings (Cmd+,):

1. **Make sure TWS / Gateway is logged in and showing a
   green / connected status.**
2. From a terminal: `nc -z 127.0.0.1 7497` (paper) or
   `nc -z 127.0.0.1 7496` (live). Should return immediately
   (no error). If `Connection refused`, TWS isn't listening on
   that port — re-check Global Configuration → API → Settings.
3. Set `broker = "ibkr-paper"` in Settings (Cmd+,). The dashboard
   should connect within ~1s; top-bar badge flips to `PAPER`.
4. F1 MAIN POSITIONS panel shows your paper account's current holdings.

If step 4 shows empty but you have positions in TWS: most
likely you logged TWS into the live account but set the
dashboard to ibkr-paper. Cross-check.

## Common errors

| Symptom | Cause | Fix |
|---|---|---|
| `Connection refused` on connect | TWS / Gateway not running, or not configured to listen on the API port | Launch + check Settings |
| `Already an API session with this client ID` | Another tool (or a previous dashboard run that didn't disconnect cleanly) is using client_id=1 | Settings (Cmd+,) → change client_id, or kill the other process |
| Disconnect during the day, can't reconnect | TWS auto-restart fired | Wait for the gateway window to show ready; manually retry from the Broker menu (Reconnect) or F6 TRADE safety panel |
| Connect succeeds but `BROKER OFFLINE` red after a few minutes | 2FA timeout / TWS session expired | Re-login to TWS; reconnect from dashboard |
| Orders accepted but never fill | Outside market hours, or your symbol's exchange isn't open | Check the order in TWS; the dashboard shows the broker's reason in F10 ORDERS |

## See also

- [`docs/live-trading-safety.md`](live-trading-safety.md) — the
  safety model the connection is wrapped in.
- Upstream:
  [IBKR API docs](https://interactivebrokers.github.io/tws-api/),
  [Quick start](https://www.interactivebrokers.com/en/trading/ib-api.php).
