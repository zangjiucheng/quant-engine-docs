# IBKR paper-trading verification

End-to-end smoke test against a real IB Gateway paper account.
Not CI-able — the test plan below assumes IBKR Gateway is
installed and you have paper credentials. Runs in five phases;
each phase has an explicit pass criterion.

## Prerequisites

- IB Gateway (recommended) **or** TWS Desktop installed.
  Gateway download: <https://www.interactivebrokers.com/en/index.php?f=16457>
- Paper account credentials (free; sign up at
  <https://www.interactivebrokers.com/en/index.php?f=46385>).
- Dashboard built from a recent main:
  ```bash
  cmake --preset=release
  cmake --build --preset=release -j
  ```

## Phase 1 — Gateway up, dashboard offline

1. Launch IB Gateway. Pick **Paper Trading**. Log in.
2. In Gateway: `Configure → Settings → API → Settings`.
   - Enable `ActiveX and Socket Clients`.
   - Set `Socket port` to **4002** (the default for paper
     Gateway).
   - Add **127.0.0.1** to `Trusted IP Addresses`.
   - Untick `Read-Only API`.
3. Confirm Gateway shows `API connection accepted` in its log
   pane after the next dashboard launch.

**Pass:** Gateway is listening on `127.0.0.1:4002` — verify with
`nc -z 127.0.0.1 4002 && echo OK`.

## Phase 2 — Dashboard connect

1. Edit `~/Library/Application Support/qe_dashboard/config.json`
   (macOS) or the XDG equivalent on Linux:
   ```json
   {
     "symbols": ["AAPL", "SPY"],
     "broker_enabled": true,
     "broker": "ibkr-paper",
     "ibkr_host": "127.0.0.1",
     "ibkr_port": 4002,
     "ibkr_client_id": 17
   }
   ```
2. Launch the dashboard.
3. Watch the top status bar.

**Pass:**
- Top bar shows `· PAPER · ibkr-paper` near the screen code.
  Hover prints `broker: ibkr-paper`.
- spdlog (stderr / Console.app) prints
  `broker: enabled (paper)` and
  `broker_session: attached ibkr-paper (paper)`.
- F2 DEPTH → ORDER TICKET stops showing the "broker disabled"
  badge.

**Fail modes:**
- Badge stays `OFFLINE`: Gateway not running, or `ibkr_port`
  wrong. Check spdlog for
  `ibkr_broker: socket connect to 127.0.0.1:NNNN failed`.
- Badge briefly LIVE-red: misconfigured `broker = "ibkr-live"`
  in config; the modal should pop. Cancel + fix config.

## Phase 3 — Submit / cancel a paper order

1. Switch to F2 DEPTH. Pick a liquid symbol (AAPL).
2. ORDER TICKET: `BUY 1 AAPL LMT 1.00` — deep out-of-the-money so
   it won't fill immediately.
3. Click **Submit**.
4. Watch WORKING ORDERS.

**Pass:**
- WORKING ORDERS shows the order with status `WORK` (Submitted /
  Accepted) within ~2 s.
- `~/Library/Application Support/qe_dashboard/orders.jsonl`
  appends rows:
  ```
  {"ts":...,"kind":"submit_attempt","broker":"ibkr-paper",...}
  {"ts":...,"kind":"submit_accepted","broker":"ibkr-paper",...,"order_id":"..."}
  ```
- Same order id appears in Gateway's `API → Orders` log pane.

5. Click **Cancel** on the working order.

**Pass:**
- Row transitions to `CANX` and falls out of the panel within
  ~2 s.
- `orders.jsonl` appends `cancel_attempt` and `cancel_accepted`
  events.

## Phase 4 — SafeBroker pre-flight + KillSwitch

1. Settings (Cmd+,) → set `max_orders_per_minute` to 2. If the
   settings panel doesn't yet expose `SafeBrokerLimits`, edit
   `config.json` directly to confirm SafeBroker default-permissive
   behavior is unchanged.
2. Press **Cmd+Shift+K**.

**Pass:**
- Red `· KILL` appears next to the screen code.
- Hover prints `kill-switch tripped — reason: user`.
- Attempting to submit a new order: error/log
  `safe_broker: kill_switch tripped (reason: user)`.
- `orders.jsonl` gains a `submit_rejected` row with the reason
  string.
- Cancel/list/account read-side still works — verify by
  cancelling the (still-open) order from phase 3 and confirming
  the working-orders list refreshes.

The kill-switch is process-lifetime; restart the dashboard to
clear it.

## Phase 5 — Reconcile drift

1. After dashboard launch, manually place an order from Gateway's
   own UI (so the dashboard's OrderLog never saw it).
2. Wait up to 30 s for the reconcile worker tick, or trigger
   immediately via `Broker → Reconnect broker` (which runs a
   fresh wrap and the next tick starts immediately).

**Pass:**
- `Broker drift detected` modal pops.
- Table lists at least one `unknown_open_order` row with the
  symbol you placed via Gateway.
- spdlog prints `reconcile_worker: drift report — N item(s)`.

Click **Close**. The dashboard logs the dismissal; the modal
won't re-pop for the same drift on subsequent ticks
(`on_drift` only fires on content change).

3. Cancel the Gateway-placed order from Gateway.
4. Wait for the next tick (or trigger via
   `Broker → Reconnect broker`).

**Pass:**
- `reconcile_worker: drift report — 0 item(s)` is logged.
  (Empty report is suppressed at the modal — we don't pop
  "all good" notifications.)

## Phase 6 — Disconnect / reconnect

1. Quit IB Gateway. Wait ~5 s.

**Pass:**
- Badge transitions from PAPER → OFFLINE.
- spdlog prints `tws_socket: read failed: ...` then
  `ibkr_connection: disconnected mid-...`.
- Submitting an order now returns
  `ibkr_connection: not connected`.

2. Relaunch IB Gateway, log in.
3. `Broker → Reconnect broker` from the menu bar.

**Pass:**
- Badge returns to PAPER within a few seconds.
- Submit / cancel works again.

## Recording results

After running all phases, capture the smoke-test record:

```bash
# Tail the order log to confirm event variety
tail ~/Library/Application\ Support/qe_dashboard/orders.jsonl

# Summarise event counts by kind:
grep -o '"kind":"[^"]*"' \
    ~/Library/Application\ Support/qe_dashboard/orders.jsonl \
    | sort | uniq -c
```

Capture the output + any failures alongside your changes before
shipping live-broker work.
