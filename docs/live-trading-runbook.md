# Live-trading runbook — operating `qe_daemon`

Step-by-step procedures for deploying, supervising, and shutting
down `qe_daemon` against a real brokerage account. Pair this with
[live-trading-safety.md](live-trading-safety.md) (the design
rationale behind every gate); this page is the operational
*how-to*.

> **Trading real money? Read [live-trading-safety.md] first.** If you
> haven't, stop. The safety model is non-negotiable context.

## What you're deploying

```
            ┌────────────────┐   journal / events
            │   qe_daemon    │ ──────────────► ~/.../state/
            │ (headless,     │
            │  launchd or    │   control socket
            │  systemd)      │ ◄────────────── qe_dashboard / qe_daemon CLI
            └────┬───┬───┬───┘
       quotes   │   │   │  orders
                ▼   │   ▼
           IBKR market data    IBKR order entry
                    │
                    └── PreTradeRisk (4 checks before broker)
```

* The daemon is the *only* process that holds an IBKR session
  in live mode. The dashboard never trades directly — it attaches
  to the daemon over the local UNIX socket.
* The state journal is the source of truth for restart-replay /
  reconcile. Don't manually edit it.
* All four risk gates are off by default; enable each in your
  `.qe` `live(...)` config.

## Putting `qe_daemon` on PATH

Every command below assumes `qe_daemon` runs from your shell —
`qe_daemon smoke`, `qe_daemon install`, `qe_daemon status`, etc.
The binary ships **inside the dashboard bundle** on macOS
(`QE Dashboard.app/Contents/MacOS/qe_daemon`) and side-by-side
with `qe_dashboard` on Linux.

Two ways to make it callable by name:

```bash
# Easy: interactive wizard. Symlinks qe_daemon (+ qe_run + qe_factor)
# into ~/.local/bin and offers to append a PATH export to your shell rc.
scripts/install-cli.sh

# Or the one-shot install + CLI symlinks in a single command:
scripts/install.sh --with-cli
```

Both commands prefer the **installed** `QE Dashboard.app` over the
dev tree, so reinstalling the app via DMG / `scripts/install.sh`
auto-propagates updates to the symlinked binary with no wizard rerun.

Manual alternative — point at whatever path the bundled binary lives at:

```bash
export PATH="/Applications/QE Dashboard.app/Contents/MacOS:$PATH"
```

## Files & locations

| Path | Purpose |
|---|---|
| `~/Library/Application Support/qe_daemon/daemon.sock` (macOS) <br> `$XDG_RUNTIME_DIR/qe_daemon.sock` (Linux) | Control socket; auth is filesystem permissions only — 0700 dir. |
| `~/Library/Application Support/qe_daemon/state/<config-stem>/journal.jsonl` | Event journal. Replayed on restart. |
| `~/Library/Logs/qe_daemon.log` / `.err` (macOS) <br> `journalctl --user -u qe-daemon` (Linux) | stdout / stderr from launchd / systemd. |
| `~/Library/Application Support/qe_daemon/logs/<config-stem>-<pid>.log` (macOS) <br> `$XDG_STATE_HOME/qe_daemon/logs/...` (Linux) | Per-run daemon log. Written by dashboard-spawned **and** terminal-launched daemons — the daemon self-bootstraps a file sink unless stdout is already redirected to a regular file (so an explicit `> file` isn't double-written). |
| `~/Library/LaunchAgents/com.jiucheng.qedaemon.plist` (macOS) <br> `~/.config/systemd/user/qe-daemon.service` (Linux) | Service template, written by `qe_daemon install`. |
| `~/Documents/quant-strategy/*.qe` | Strategy configs. The `live(...)` wrapper is what makes one runnable by the daemon. |

## Writing a `live(...)` config

A daemon-runnable `.qe` evaluates to a `live(...)` value. The base
is a `backtest(...)` block — byte-identical to what you'd run
offline — wrapped with the live-only knobs.

```qe
let base = backtest(
  data = ibkr_stock("SPY"),
  strategy = signal(
    entry = cross_above(sma(close, 20), sma(close, 50)),
    exit  = cross_below(sma(close, 20), sma(close, 50)),
    symbol = "SPY",
    size   = 100,
  ),
  execution = execution(
    capital        = 100_000,
    commission_bps = 0.5,
    slippage_bps   = 1.0,
    fill_model     = "next_open",
  ),
)

live(
  base   = base,
  broker = "ibkr-paper",                 # always start here
  symbols = ["SPY"],                     # superset of data symbols allowed
  journal_dir = "/Users/me/.qe/state/spy_cross",

  # IBKR Gateway connection (optional — all fields have defaults).
  # Override host / port / client_id when running multiple daemons
  # or a non-standard TWS/Gateway layout.
  ibkr = ibkr_connection(
    host      = "127.0.0.1",
    port      = 7497,                    # 7497 paper TWS, 4002 paper Gateway,
                                         # 7496 live TWS, 4001 live Gateway
    client_id = 1,                       # bump if another QE binary uses 1
    account   = "",                      # "" = daemon picks first available
  ),

  # Pre-trade risk — every limit defaults off.
  # Set the four that matter for your strategy size.
  risk_max_position_per_symbol = 200,    # shares (absolute value)
  risk_max_gross_exposure_usd  = 60_000, # Σ |price · pos|
  risk_max_daily_loss_usd      = -1500,  # NEGATIVE; trips kill-switch
  risk_max_orders_per_minute   = 10,
)
```

The `ibkr` kwarg defaults so a minimal `live(base, broker, symbols)`
config still connects to a local paper Gateway. The daemon picks
`port = 7497` when `broker = "ibkr-paper"` and `7496` when
`broker = "ibkr-live"` — override only if you're running a non-
default Gateway port or need a stable `client_id`.

### Multi-leg + cross-sectional (`signalize_universe`)

The daemon's live engine handles N-leg portfolios — including the
cross-sectional shape `signalize_universe(...)` produces. Each
leg in `cfg.base.strategy.strategies` gets its own evaluator pair
and position state; the engine subscribes to the union of all
leg `trade_symbol`s (plus any reference feeds in `cfg.symbols`).

For cross-sectional strategies that use `is_top` / `is_bottom` /
`quantile`, the engine uses a **fork-join barrier** at every bar
close (EPIC-69). Without the barrier, the first universe symbol
to close at a new timestamp would trigger the prepass against
1 fresh + (N-1) stale slots — picking arbitrary "winners" by
arrival order. With the barrier:

- `on_bar` for each leg's symbol records the bar but **defers**
  prepass + leg evaluation.
- When every universe symbol has produced a bar for the current
  `close_ts_ns` (quorum), the engine flushes once: refresh
  `per_symbol_ctxs_`, prepass every leg, evaluate, submit orders.
- If a new `close_ts_ns` arrives while a prior batch is still
  pending, the prior batch is force-flushed with whatever it has
  (`reason=new_ts`) so the rebalance window isn't lost.
- If a 5 s wall-clock elapses without quorum (one or more
  symbols never ticked that interval), the batch flushes with
  NaN-filled stale slots (`reason=timeout`, WARN log). The
  prepass correctly refuses to fire trades on NaN-valued symbols
  — half-formed ranks don't drive orders.

Non-cross-sectional portfolios bypass the barrier entirely
(per-bar fast path).

### Daily-resolution strategies

For `yahoo_template("1d", ...)` strategies (typical
`signalize_universe` shape), the daemon's bar aggregator uses
`BarAggregator::kDailyResolutionNs` — bars close at session close
(16:00 ET) instead of every N nanoseconds. All universe symbols'
daily bars close within seconds of each other, so quorum is
reached and the rebalance fires in one batch around the close.
Orders submitted at close execute at next morning's open (DAY
order convention).

### Picking risk limits

A useful starting heuristic, **for a single-strategy / single-symbol
deploy**:

* **per-symbol position**: 1.2 × max position the backtest ever
  held. Catches a runaway loop without rejecting legitimate fills.
* **gross exposure**: 1.5 × `execution.capital`. The same multiple
  works for short books because the check uses `|pos · price|`.
* **daily loss**: roughly the 95th percentile of historical
  drawdowns in your backtest, NEGATIVE.
* **orders/minute**: 10 is comfortable for a daily/intraday strategy.
  Drop to 3 for once-a-day rebalances; raise to 60 if you're doing
  high-touch market making.

When in doubt: start tighter. The cost of a false reject is the
order doesn't go out (the strategy will re-fire next bar). The
cost of a false negative is a runaway loop on a real account.

## Deployment — first time, paper

1. **Verify the strategy ran offline.** `qe_run path/to/base.qe`
   should produce a `results.json`. The daemon will refuse to
   start on a config it can't parse.

2. **Build a `live_*.qe` from the working `backtest(...)`.** Wrap
   it as shown above. Set `broker = "ibkr-paper"` and conservative
   risk limits.

3. **Start the IBKR Gateway in paper mode.** Verified against
   IB Gateway 151+ on port 4002 (the default IB Gateway paper
   port) and TWS Desktop on port 7497. The daemon won't auto-
   launch it. Before the first connect, walk the Gateway settings:
   - `Configure → API → Settings → Enable ActiveX and Socket
     Clients` is ON.
   - `Trusted IPs` includes `127.0.0.1`.
   - `Read-Only API` is off (otherwise order submission will
     fail at the broker layer).
   - `Master API client ID` is empty.

4. **Foreground smoke**:
   ```
   qe_daemon start ~/Documents/quant-strategy/live_spy.qe --log-level=info
   ```
   You want to see (in order):
   - `config parsed OK — kind = live`
   - the four risk limits echoed
   - `broker session up — name = ibkr-paper`
   - `data-path IbkrConnection constructed`
   - `IBKR handshake OK — starting quote subscriptions`
   - `historical warmup = N bars for SPY` (N = 60 on a 60 D /
     1 day warmup)
   - `disconnect_watchdog: armed`
   - `live_engine: started — broker=ibkr-paper, trade_symbol=SPY`

   For the full operator-verified output (including the
   `Market data farm connection is OK:usfarm` messages from
   IBKR and the `Requested market data is not subscribed`
   warning paper accounts always get), see
   [`qe-daemon-smoke.md`](qe-daemon-smoke.md).

   `Ctrl-C` once you're confident the wire-up is right.

   > **`client_id` collisions.** The daemon opens TWO IBKR
   > connections — a data path (the `.qe` value `N`) and a
   > broker path (`N + 1`). Default `N = 1` uses `1` + `2`,
   > which collides with the dashboard's default `client_id = 1`.
   > Set `ibkr = ibkr_connection(client_id = 17)` (or whatever
   > distinctive id) in any `.qe` you'll run alongside the
   > dashboard.

5. **Install under launchd / systemd**:
   ```
   qe_daemon install ~/Documents/quant-strategy/live_spy.qe
   ```
   The CLI writes the plist or unit, then loads it. Check it's
   running:
   ```
   qe_daemon status
   ```
   Returns a JSON object the dashboard's DAEMON chip mirrors.

6. **Attach the dashboard** (any time after step 5). The top-bar
   DAEMON chip flips green within ~2 s. F6 TRADE's safety panel
   reads:

   ```
   SAFETY · kill: armed · reconcile: clean · broker_session: PAPER
          · daemon: attached · broker: connected
   [Trip kill-switch]  [Reconnect broker]  [Re-reconcile now]
   [Stop daemon]
   ```

   While the daemon owns the broker, the four F6 data panels
   (Working Orders / Executions / Account / Order Log) read from
   the daemon's control socket — `DaemonOrderCache` polls
   `orders` / `positions` / `equity` / `log_tail` every 3 s and
   reshapes the responses into the same `OrderSnapshot` the
   local-broker panels consume. So you see the daemon's live
   blotter in the dashboard even though the dashboard itself
   isn't connected to IBKR. (EPIC-66/67)

   Two badges to scan for:

   - **`daemon:`** — control-socket attach state. `attached`
     (green) means the dashboard can read state + send commands.
     `disconnected` (amber) means the dashboard saw the socket
     go away; the auto-poll re-attaches when the daemon comes
     back. `—` means no daemon is running.
   - **`broker:`** — the daemon's view of its broker link
     (EPIC-70). `connected` (green) is healthy.
     `paused · Ns` (amber) means the disconnect watchdog tripped
     the soft-pause — new orders reject for the duration, open
     orders untouched. `TRIPPED (reason)` (red) means the
     watchdog escalated to the hard trip; operator must restart.

7. **Tail events** while you watch the first session:
   ```
   qe_daemon tail events
   ```

## Going live — additional gates

`broker = "ibkr-live"` flips two things:

* The daemon won't start without the env gate `QE_LIVE_TRADING=1`.
* The dashboard's broker chip paints red `LIVE`. The kill-switch
  modal is one keystroke away (`Cmd-Shift-X`).

> Practice tripping the kill-switch in paper before you go live.
> You should know what the dashboard looks like in the OFF state
> before you ever need to use it under pressure.

The promotion procedure:

1. Paper-deploy with the same `.qe` for **at least 5 sessions**
   with no manual interventions or unexpected events in the tail.
2. Flatten any positions at IBKR-paper.
3. Edit the config: `broker = "ibkr-live"`. Re-run risk-limit
   sanity check.
4. `qe_daemon uninstall` to drop the paper service.
5. `QE_LIVE_TRADING=1 qe_daemon install <live_config.qe>`.
6. Watch `qe_daemon tail events` for the first hour.

## Day-to-day operations

| Need to | Run |
|---|---|
| Check daemon is up | `qe_daemon status` or just look at the F6 `daemon:` badge |
| Watch live events | `qe_daemon tail events` or F6 ORDER LOG panel (reads from the daemon's `log_tail` verb) |
| Watch a specific channel | `qe_daemon tail <channel>` |
| Stop the daemon cleanly | **F6 TRADE → "Stop daemon" button** (preferred), or `qe_daemon stop`, or `launchctl bootout` / `systemctl --user stop qe-daemon` |
| Restart after a config edit | F6 → Stop daemon → re-Deploy from F6 |
| Deploy a new `.qe` from the dashboard | F3 WKSP → Cmd+S on a `live(...)` file (registers it, doesn't auto-deploy) → F6 TRADE → Deploy panel → "Arm deploy" → "Click again to deploy" |
| Recover a missed close eval (same evening) | `qe_daemon backfill --latest`, or `qe_daemon backfill <close_ts_ns>` for a specific close — see "A close eval was missed" below |
| Trip the kill-switch from CLI | `qe_daemon kill <reason>` *(wire-through pending — see "What's still in flight" below)* |
| Trip from the dashboard | F6 TRADE → "Trip kill-switch", or `Cmd-Shift-X` |
| Uninstall the LaunchAgent / unit | `qe_daemon uninstall` |

### F6 TRADE Deploy panel (EPIC-66)

When no daemon is attached, F6's top half becomes a Deploy panel
that lets you launch `qe_daemon` without dropping to a terminal:

1. F3 WKSP — open the `live(...)` `.qe` file and `Cmd+S`. The
   workspace registers it as `cfg.active_live_path` and drops
   a red `[LIVE]` badge next to it in the file tree. **Cmd+S
   on a `live(...)` file does NOT auto-start the daemon** — the
   two-click gate lives on F6.
2. Switch to F6. The Deploy panel previews the registered file:
   broker, IBKR endpoint, symbol count, capital, paper-vs-live
   mode.
3. Click **"Arm deploy"**. The button label changes to "Click
   again to deploy" and arms for 5 s.
4. Click again. The dashboard double-forks + `setsid`'s the
   daemon (so it survives the dashboard exiting), redirects
   `stdout` / `stderr` to
   `~/Library/Application Support/qe_daemon/logs/<stem>-<pid>.log`,
   and starts the attach poll. Within ~2 s the daemon's control
   socket comes up and the F6 chips flip to attached + connected.
5. For live (non-paper) brokers, the panel additionally requires
   you to type the broker name into a confirm box before the
   "Click again to deploy" button enables. Same gesture, extra
   pause for real money.

If the daemon exits within seconds of spawn, the Deploy panel
polls `kill(pid, 0)` each frame; once the process is gone it
reads the log file's trailing `[error]` lines and renders them
inline, so you don't have to dig through logs to see why a
deploy failed.

> **Never edit `journal.jsonl` by hand.** The restart-replay
> reconciler reads it; an out-of-shape line will fail loudly but
> a *plausible* edit can silently desync state.

## When something goes wrong

### Daemon says it can't reach the broker

The disconnect watchdog (EPIC-70) is a 3-state machine: **Healthy
→ Paused → Tripped**. What you see depends on how long the
broker has been offline:

| Wall-clock offline | Watchdog state | F6 `broker:` badge | Effect |
|---|---|---|---|
| ≤ 3 s | Healthy | `connected` | Normal — TCP heartbeat blip absorbed |
| 3 – 30 s | **Paused** | `paused · Ns` (amber) | New orders reject (`PreTradeRisk` gate). Open orders untouched. Auto-resumes on reconnect. |
| > 30 s | **Tripped** | `TRIPPED (ibkr_disconnect)` (red) | KillSwitch tripped, open orders cancelled. Operator must restart. |

For a transient outage (Gateway restart, brief wifi drop), the
daemon auto-recovers — no action needed. Log lines:

```
[warn]  disconnect_watchdog: broker offline for 3 polls — entering
        soft-pause (trip threshold 30 polls)
[warn]  pre_trade_risk: watchdog pause armed — new orders will
        reject until broker reconnects
... gateway comes back ...
[info]  disconnect_watchdog: broker reconnected after 10 misses
[info]  pre_trade_risk: watchdog pause cleared — order submission
        resumed
```

For a sustained outage (badge stays `TRIPPED`):

1. Is the IBKR Gateway up? Check the Gateway UI.
2. Is anything else holding the TWS session? Only one client per
   account.
3. F6 → Stop daemon (or `qe_daemon stop`) and restart the
   Gateway first, then redeploy.

### A close eval was missed (red `eval_missed` alert)

The F6 EVAL line shows a red `N alerts` chip, or
`qe_daemon tail events` shows an `eval_missed` row. The usual
cause is a Gateway flap around the close (16:01 ET bounce eats
the close bar). Two recovery paths:

- **Automatic** — if the miss was connectivity-induced, the
  daemon backfills on its own when TWS raises `1101`/`1102`
  ("connectivity restored"): it replays the missed eval against
  its bar history and submits the recovered orders. Look for
  `reconnect-triggered backfill status=ok` in the log and an
  `eval_backfilled` journal row.
- **Manual** — same evening (within 8 h of the close):

  ```bash
  qe_daemon backfill --latest        # newest eval_missed
  qe_daemon backfill <close_ts_ns>   # a specific close
  ```

  Exit 0 means `ok` or `already_handled` (the live eval or a
  prior backfill got there first — safe no-op); exit 3 means it
  refused (`out_of_window`, `insufficient_bars`, ...) and the
  response JSON says why.

Backfill is idempotent — re-running it, or racing the automatic
trigger, never double-submits. If the alert is NOT
connectivity-shaped (strategy threw, `eval_replay_mismatch`),
fix the signal first; details in
[Eval self-healing](eval-self-heal.md).

### Daemon stopped on its own

`tail ~/Library/Logs/qe_daemon.err`. Three common causes:

- Pre-trade `daily-loss kill` tripped → expected; restart only
  after you investigate the equity drop.
- Reconcile-vs-broker drift at startup → broker positions don't
  match the journal. **Don't auto-fix.** The daemon refuses to
  trade and points you at a drift report. Either flatten at the
  broker or rotate the journal under `--reset-journal`.
- Crash → the LaunchAgent / unit relaunches automatically
  (`KeepAlive: Crashed=true` / `Restart=on-failure`). Check
  the journal stuck point.

### The dashboard's DAEMON chip is amber

Reader thread saw EOF since last attach — daemon dropped or got
SIGTERM'd. The dashboard retries every ~2 s. If it stays amber:

```
qe_daemon status
```

If that errors with "no daemon running": something killed it —
check service status and logs.

### A risk gate rejected an order I expected to clear

The journal records every rejection with its reason. Tail
`events` (filter for `kind == "decision"` with `accepted == false`)
or grep the journal. The four reasons:

- `per-symbol position cap`
- `gross exposure cap`
- `daily loss kill` (also tripped the switch)
- `orders-per-minute throttle`
- `kill-switch tripped`

**Don't widen a limit "for one trade".** Either the limit was
miscalibrated (raise it permanently after a deliberate review)
or the strategy is misbehaving (fix the strategy).

## What ships, what doesn't

The end-to-end wire-up landed (event loop → router → risk →
journal → socket → IBKR), and the `epic/62-finish` follow-ups
closed the remaining EPIC-62 acceptance criteria except for
two operator-side gaps:

**Shipped:**

- **IBKR `reqHistoricalData` warmup** — the daemon calls
  `IbkrConnection::request_historical_bars(sym, "60 D", "1 day")`
  for **every** subscribed symbol after the handshake (EPIC-66
  follow-up). Pre-fix only the first symbol was warmed, which
  made cross-sectional deployments start with NaN factor values
  for 29 of 30 symbols. Now logs as
  `historical warmup = N bars across M symbols (K ok, F failed)`;
  per-symbol failures are non-fatal.
- **3-state disconnect watchdog** — `qe::live::DisconnectWatchdog`
  (EPIC-70) polls `IbkrConnection::is_connected()` every 1 s. After
  ≥ 3 consecutive misses it enters **Paused**: fires
  `on_soft_pause("ibkr_disconnect_soft")` → `PreTradeRisk::
  set_watchdog_pause` → new orders reject; open orders untouched.
  Reconnect within the 30-tick trip window auto-resumes
  (`on_resume` → `clear_watchdog_pause`). At 30 consecutive misses
  the watchdog escalates to **Tripped** — `KillSwitch.trip
  ("ibkr_disconnect")` + cancel every open broker order. The hard
  trip is sticky for the process lifetime; restart the daemon to
  clear.
- **Broker reconciliation on restart** —
  `qe::live::reconcile_positions()` compares the journal-replayed
  `in_position[symbol]` map against
  `broker_session->list_positions()` and reports per-symbol drift.
- **F6 TRADE daemon-mode panels** (EPIC-67) — Working Orders /
  Executions / Account / Order Log read from the daemon's control
  socket (`orders` / `positions` / `equity` / `log_tail`) when a
  daemon is attached. `DaemonOrderCache` polls every 3 s and
  reshapes the JSON into the same `OrderSnapshot` shape the
  local-broker panels consume. Cancel button is disabled in
  daemon mode (use Trip kill-switch to cancel all open orders).
- **F6 Deploy panel + Stop daemon button** (EPIC-66) — see the
  "F6 TRADE Deploy panel" section above. F6 also has a
  "Stop daemon" button next to Trip kill-switch / Reconnect
  broker / Re-reconcile that sends the `stop` control verb for
  a graceful shutdown.
- **Cross-sectional fork-join barrier** (EPIC-69) — see the
  "Multi-leg + cross-sectional" section above.
- **Daily-resolution bar aggregator** (EPIC-66) — daily strategies
  close their bar at 16:00 ET (session close) instead of every N
  nanoseconds; required for `yahoo_template("1d", ...)`
  deployments to evaluate correctly.
- **Staged entry / exit** (EPIC-74) — when a `.qe` `execution(...)`
  block carries an `entry_schedule = staged_entry(...)` or
  `exit_schedule = staged_exit(...)`, the daemon attaches an
  `OrderScheduler` that expands each strategy intent into 2-4
  time-windowed slices and dispatches them across 1-2 trading
  sessions. Same `SliceSchedule::expand()` used in backtest, so
  walk-forward PnL reflects the live fill sequence. Slice state
  (`slice_scheduled` / `slice_fired` / `slice_cancelled` events)
  is journaled; restart replays the journal to resume the queue
  without double-submitting fired slices. Kill-switch trip
  cancels every Pending slice. See
  [Staged entry / exit](staged-entry.md) for the full reference
  including the F6 TRADE STAGED panel.
- **Eval self-healing** (EPIC-75) — every close eval journals a
  canonical `eval_round`; same-close re-evals are idempotent
  (never double-order); a background sweep alerts on missed
  closes (`eval_missed`); and a missed close can be **backfilled
  same-evening** — automatically on broker reconnect (TWS
  `1101`/`1102`) or manually via `qe_daemon backfill --latest`.
  F6 TRADE surfaces the latest round + alert count on the EVAL
  line. See [Eval self-healing](eval-self-heal.md).

**Not yet wrapped:**

- **`kill` / `unkill` / `pause` / `resume` CLI subcommands** —
  the verbs work over the raw socket (the dashboard's F6 TRADE
  panel uses them), but `qe_daemon kill` / `qe_daemon pause`
  from a shell aren't wrapped yet. Trip from the dashboard
  (F6 button or `Cmd-Shift-X`) or send `SIGTERM` for a clean
  shutdown.
- **Per-order cancel from the dashboard in daemon mode** — the
  daemon doesn't yet expose a `cancel_order` control verb, so the
  F6 Working Orders panel's Cancel button is disabled when in
  daemon mode (tooltip points at Trip kill-switch as the working
  alternative). Trip kill-switch cancels ALL open orders.
- **Alpaca daemon support** — only `ibkr-paper` / `ibkr-live`
  start the runtime. `broker = "alpaca-..."` parses but exits 4
  with a clear error.

See [`qe-daemon-smoke.md`](qe-daemon-smoke.md) for the manual
IBKR-Gateway smoke checklist you should run after first install
and again before any live promotion. The smoke was verified
end-to-end against IB Gateway 151 paper on 2026-06-04.

[live-trading-safety.md]: live-trading-safety.md
