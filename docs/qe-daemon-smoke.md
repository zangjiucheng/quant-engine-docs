# `qe_daemon` IBKR-paper smoke checklist

Manual end-to-end verification that the wired daemon
(EPIC-62 T62.4 / T62.5 / T62.6 / T62.11 + the
`epic/62-finish` follow-ups) connects, warms up, ticks, and
stops cleanly against a real IB Gateway. Not gated by `ctest`
— the daemon's own unit tests cover the modules in isolation.

This page was rewritten after running the smoke against IB
Gateway 151+ on 2026-06-04; the quoted log lines below are
verbatim from that run.

## Prerequisites

1. **IB Gateway running in paper mode**. The daemon supports both
   IB Gateway and TWS Desktop — pick the right port:

   | App                    | Port  |
   |------------------------|-------|
   | Paper IB Gateway       | 4002  |
   | Paper TWS Desktop      | 7497  |
   | Live IB Gateway        | 4001  |
   | Live TWS Desktop       | 7496  |

   Log in to the Gateway / TWS with your paper account. Confirm:
   - `Configure → API → Settings → Enable ActiveX and Socket
     Clients` is **on**.
   - `Trusted IPs` includes `127.0.0.1` (and your dashboard
     machine's LAN address if remote-attaching).
   - `Read-Only API` is **off** — otherwise order submission
     fails at the broker layer (`status` and `positions` still
     work).
   - `Master API client ID` is left empty.

2. **Build**: `cmake --build --preset=release -j`.

3. **client_id budget**. The daemon opens TWO concurrent IBKR
   connections from one process — a data path (the `.qe`'s
   `ibkr_connection(client_id = N)` value) and a broker /
   order-routing path (`N + 1`). Pick an `N` such that neither
   `N` nor `N + 1` collides with your dashboard, any TWS
   bookkeeping client, or a second daemon. The default `N = 1`
   uses `1` and `2`; smoke fixtures in this guide use `N = 0`
   so they don't fight the dashboard's `client_id = 1` default.

## The smoke fixture

A minimal `live(...)` config pointing at the paper IB Gateway:

```qe
let base = backtest(
  data = file("tests/fixtures/csv/short_60days.csv"),
  strategy = signal(
    entry  = cross_above(sma(close, 5), sma(close, 20)),
    exit   = cross_below(sma(close, 5), sma(close, 20)),
    symbol = "SPY",
  ),
  execution = execution(capital = 100_000),
)

live(
  base    = base,
  broker  = "ibkr-paper",
  symbols = ["SPY"],
  ibkr    = ibkr_connection(host = "127.0.0.1", port = 4002, client_id = 0),
)
```

Save to `/tmp/qe_smoke.qe` (or anywhere). Adjust `port` to
`7497` for TWS Desktop.

## Cold start

```bash
./build/release/bin/qe_daemon start /tmp/qe_smoke.qe
```

Expected log sequence (verbatim from the verified run):

```
daemon: connecting to IBKR Gateway at 127.0.0.1:4002
daemon: broker session will connect with client_id=1 (data uses 0)
ibkr_connection: TWS msg [2104, req=-1]: Market data farm connection is OK:usfarm
ibkr_connection: TWS msg [2107, req=-1]: HMDS data farm connection is inactive but should be available upon demand.ushmds
ibkr_connection: TWS msg [2158, req=-1]: Sec-def data farm connection is OK:secdefil
daemon: broker session up — name = ibkr-paper
daemon: opening data-path connection to 127.0.0.1:4002
daemon: data-path IbkrConnection constructed, client_id=0
daemon: journal dir = ~/Library/Application Support/qe_daemon/state
daemon: cold start (no prior journal events)
daemon: control socket listening at ~/Library/Application Support/qe_daemon/daemon.sock
daemon: IBKR handshake OK — starting quote subscriptions
daemon: requesting historical warmup for SPY (60 D, 1 day)
ibkr_connection: TWS msg [2106, req=-1]: HMDS data farm connection is OK:ushmds
daemon: historical warmup = 60 bars for SPY
daemon: entering live event loop
disconnect_watchdog: armed (poll=1000ms, tolerance=5 ticks ~= 5000ms)
ibkr_quote_source: subscribed SPY → ticker_id=10000000
live_engine: started — broker=ibkr-paper, trade_symbol=SPY, subscribed=1 symbol(s), resolution=1m
```

The two `TWS msg [2104/2107/2158]` blocks each appear twice
because both the broker and data-path connections receive the
farm-connection notices independently. That's expected.

> **Paper-account quirk.** After subscribe you may see
> `TWS error [10167, ...]: Requested market data is not subscribed.
> Displaying delayed market data.` — that's IBKR telling us a
> paper account without a paid real-time data subscription will
> get delayed tick types (74 / 75 / 76 — delayed volume / close /
> open). The daemon parses these as informational; **no bars close
> outside RTH** because delayed-summary types aren't trade ticks.
> To exercise bar closes during the smoke you need either
> regular trading hours, a paid market-data subscription, or
> running against TWS with replay enabled.

## Control socket round-trip

In a second shell, while the daemon runs:

```bash
./build/release/bin/qe_daemon status
```

Expected:

```json
{
  "bars_processed": 0,
  "broker": "ibkr-paper",
  "events_written": 0,
  "kill": false,
  "orders_submitted": 0,
  "paused": false,
  "uptime_s": 18
}
```

```bash
./build/release/bin/qe_daemon tail events
```

(Stays idle outside RTH — every journal append flows through
the `events` push channel; with no bars closing yet, none have
been written. Inside RTH or after a manual position change you
should see one JSON line per event.)

## Graceful stop

```bash
./build/release/bin/qe_daemon stop
```

Expected:

```
ok: {"stopping":true}
```

And in the daemon log:

```
control: stop — flagging shutdown
live_engine: stopped — 0 bars processed, 0 orders submitted
daemon: shutdown complete (bars=0, orders=0, journal_events=0)
```

Exit code `0`.

## Verifying the disconnect watchdog

The watchdog is a 3-state machine since EPIC-70: **Healthy →
Paused → Tripped**. The smoke tests both transitions.

### Soft-pause + auto-resume (EPIC-70)

While the daemon is running, **stop the API listener inside
Gateway** (Configure → API → Settings → uncheck "Enable
ActiveX and Socket Clients", then OK). Within ~3 seconds the
daemon log prints:

```
[warn]  disconnect_watchdog: broker not connected (1/30 consecutive misses, soft-pause at 3)
[warn]  disconnect_watchdog: broker not connected (2/30 consecutive misses, soft-pause at 3)
[warn]  disconnect_watchdog: broker offline for 3 polls — entering soft-pause (trip threshold 30 polls)
[warn]  pre_trade_risk: watchdog pause armed — new orders will reject until broker reconnects
```

A `qe_daemon status` from a separate shell now returns
`"watchdog_state": "paused"`, `"broker_connected": false`,
`"paused_by_watchdog": true`,
`"pause_reason": "ibkr_disconnect_soft"`. F6's `broker:` badge
flips amber: `paused · Ns`.

**Re-enable the API listener within 30 seconds.** The watchdog
catches it on the next poll and resumes:

```
[info]  disconnect_watchdog: broker reconnected after 4 consecutive misses
[info]  pre_trade_risk: watchdog pause cleared — order submission resumed
```

`watchdog_state` returns to `"healthy"`, F6 `broker:` badge
returns to green `connected`. **No daemon restart needed.**

### Hard trip after sustained outage

Repeat the test but leave the API listener disabled past the 30 s
trip threshold. After 30 polls:

```
[error] disconnect_watchdog: broker offline for 30 consecutive polls — tripping kill-switch + invoking on_trip
[warn]  watchdog: cancelled N open order(s) after kill-switch trip
```

`qe_daemon status` now returns
`"kill": true, "kill_reason": "ibkr_disconnect"`,
`"watchdog_state": "tripped"`. F6 `broker:` badge is red
`TRIPPED (ibkr_disconnect)`. Re-enabling the API does **NOT**
auto-unkill — `KillSwitch` is one-way for safety. Stop the
daemon (F6 → Stop daemon or `qe_daemon stop`) and redeploy to
clear.

## Verifying broker reconcile

1. Run the smoke; let it process at least one journal event
   (a position transition, or trigger one via the dashboard's
   F6 TRADE panel).
2. `qe_daemon stop`.
3. **From the Gateway UI**, flatten the position or change
   its quantity.
4. Re-launch the daemon. Expected log:
   ```
   daemon: broker reconcile found 1 drift row(s) — operator review recommended before placing new orders. Full report:
   broker_reconcile: 1 DRIFT row(s) (journal=1 symbols, broker=0 symbols)
     SPY  journal=100  broker=0  Δ=-100  [ghost — journal has it, broker doesn't]
   ```
5. The same row also lands in the journal as a `Notice` event
   (visible via `qe_daemon tail events`).

## Restart-recovery check

```bash
# 1. Run for at least one journal-event-producing window.
./build/release/bin/qe_daemon start /tmp/qe_smoke.qe

# 2. Inspect the journal directly.
cat ~/Library/Application\ Support/qe_daemon/state/events.jsonl | tail

# 3. Stop + restart. Expect:
#    "daemon: replayed N journal events; restored M positions,
#     bar_count=..., last_equity=..."
./build/release/bin/qe_daemon stop
./build/release/bin/qe_daemon start /tmp/qe_smoke.qe
```

The broker-reconcile pass runs automatically on every restart
after the IBKR handshake. Drift-clean runs log a one-liner:
`daemon: broker reconcile clean (journal=N sym, broker=N sym)`.

## Verifying the cross-sectional barrier (EPIC-69)

Only meaningful when the smoke `.qe` is a `signalize_universe`
(or otherwise uses `is_top` / `is_bottom` / `quantile` /
`rank`). For a single-symbol smoke this section is a no-op.

The barrier shows up in the daemon log as `[xs_batch:reason]`
suffixes on every ENTRY/EXIT line. Three reasons:

| `reason` | When | Operator action |
|---|---|---|
| `quorum` | Every universe symbol produced a bar for the current `close_ts_ns`. The expected, healthy case. | None |
| `new_ts` | A bar at a fresh `close_ts_ns` arrived while the prior batch was still pending (some symbol never ticked at the prior ts). Prior batch force-flushed with stale slots. | Investigate why a symbol skipped a bar — usually low liquidity. |
| `timeout` | 5 s wall-clock elapsed since the batch started without quorum. WARN-logged. | Same as `new_ts` — check the missing symbol's tick stream. |

What `quorum` looks like for a 30-symbol cross-sectional close:

```
[info] live_engine: ENTRY AAPL (AAPL) qty=1 @ ts_ns=... [xs_batch:quorum]
[info] live_engine: ENTRY MSFT (MSFT) qty=1 @ ts_ns=... [xs_batch:quorum]
... up to 10 entries (top_k=10) all at the same ts_ns ...
```

All `K=top_k` entries land in one flush at the same
`close_ts_ns`. The prepass computed ranks against fresh data
for every universe symbol — orders reflect the **actual**
top-K, not an arrival-order race.

What `timeout` looks like:

```
[warn] xs_barrier: timed out waiting for 27/30 symbols at ts_ns=... — dispatching with stale slots (rank quality degraded)
[info] live_engine: ENTRY AAPL (AAPL) qty=1 @ ts_ns=... [xs_batch:timeout]
```

The 3 missing symbols' contexts fall back to whatever
`last_ctx` had (prior bar's close, or NaN if never observed).
The prepass returns 0 for NaN slots in `is_top` / `is_bottom`
so half-formed ranks don't fire trades against NaN-valued
symbols — `timeout` flushes that don't produce any entry log
lines are working as designed (defensive bail-out).

## Staged entry / exit smoke (EPIC-74)

Run when the `.qe` you're deploying carries
`entry_schedule = staged_entry(...)` (or `exit_schedule`).

1. **Daemon log mentions the scheduler.** On startup, look for:

   ```
   [info] order_scheduler: enabled (entry slices=4, exit slices=0)
   ```

   If the log says nothing, the schedule is unset on this `.qe`.

2. **Strategy intent expands into N journal rows.** After a signal
   fires, `grep slice_scheduled events.jsonl` should show one row
   per slice with `parent_signal_id`, `slice_idx`, `slice_total`,
   `fire_at_ns`, and `signed_qty`.

3. **F6 TRADE STAGED panel populates.** The WORKING ORDERS panel
   grows a "STAGED ORDERS" section listing each pending slice with
   its symbol, side, slice index, qty, state, and fire time. Until
   any slice fires the state column reads `pending` (green).

4. **First slice fires at its window.** When wall-clock crosses
   the first slice's `fire_at_ns`, the daemon log emits:

   ```
   [info] order_scheduler: fired 1 slice(s) at ts_ns=...
   ```

   A matching `slice_fired` row appears in the journal, the
   broker submits the order through the same risk-gated router as
   un-staged orders, and the panel's first row flips to `fired`
   (dim).

5. **Restart recovery.** Kill the daemon (`Cmd-Shift-X` or
   `qe_daemon stop`) after 1-2 of N slices have fired. Restart the
   daemon and confirm the log line `order_scheduler: hydrated N
   slice row(s) from journal`. The STAGED panel resumes with the
   same fired/pending split. Subsequent ticks fire ONLY the
   un-fired slices — the broker should NOT see a re-submission of
   the slices that fired pre-crash.

6. **Kill-switch cancels Pending.** Trip the kill-switch with at
   least one slice still Pending. The watchdog log emits
   `watchdog: cancelled N pending staged slice(s)` and the panel's
   pending rows flip to `cancel` (red). No broker submissions
   occur for cancelled slices.

If step 5 fails — the daemon re-submits a slice the broker already
filled pre-crash — STOP. That's a double-submit, not a smoke nit;
file an issue with the relevant journal segment + IBKR paper
account ids.

## Known limitations (NOT smoke gates)

- **Bar closes during the smoke require live ticks.** Paper
  accounts without a paid real-time data subscription get
  delayed-summary tick types that the bar aggregator drops as
  non-trades. To exercise bar closes + strategy fires, either:
  use the smoke during RTH, subscribe to live market data on
  the paper account, or wire a tick-replay source for offline
  testing.
- **`qe_daemon pause` / `qe_daemon kill` CLI subcommands** —
  the verbs work over the raw socket (the dashboard's F6 TRADE
  panel uses them), but `qe_daemon pause` from a shell isn't
  wrapped yet. Use the dashboard or post the JSON request
  directly: `{"op":"pause"}` length-prefixed to the socket.
- **Multi-symbol live** — `LiveEngine` is single-strategy /
  single-trade-symbol. Multi-leg `portfolio(...)` configs
  return `Status::Unsupported` cleanly.
- **Alpaca daemon support** — `broker = "alpaca-..."` exits
  code 4 with a clear error. Daemon ships ibkr-paper /
  ibkr-live only.
