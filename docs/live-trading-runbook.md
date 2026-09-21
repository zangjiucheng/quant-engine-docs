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
  `.qe` `live(...)` config. **`broker = "ibkr-live"` refuses to
  start unless all four are armed** (EPIC-88 T88.6) — the refusal
  happens before the daemon opens a socket to the venue. A
  `-paper` broker only warns.

## Putting `qe_daemon` on PATH

Every command below assumes `qe_daemon` runs from your shell —
`qe_daemon smoke`, `qe_daemon install`, `qe_daemon status`, etc.
The binary ships **inside the dashboard bundle** on macOS
(`QE Dashboard.app/Contents/MacOS/qe_daemon`) and side-by-side
with `qe_dashboard` on Linux.

Use [CLI setup](install.md#using-the-cli-tools-from-a-terminal) for a
DMG installation. From a source checkout, `scripts/install-cli.sh` or
`scripts/install.sh --with-cli` creates the symlinks.

## Files & locations

| Path | Purpose |
|---|---|
| `~/Library/Application Support/qe_daemon/daemon.sock` (macOS) <br> `$XDG_RUNTIME_DIR/qe_daemon.sock` (Linux) | Control socket; auth is filesystem permissions only — 0700 dir. |
| `~/Library/Application Support/qe_daemon/state/events.jsonl` | Event journal. Replayed on restart. **Flat, not per-config** — every deploy that doesn't set `journal_dir` shares this one file. Rows carry a `strategy_id` so the scheduler can tell them apart, but running two configs against one journal is still a bad idea: use `journal_dir = "..."` per deploy. Rotated segments sit beside it as `events-<UTC>.jsonl` and are part of the history — see [Journal rotation](#journal-rotation). |
| `~/Library/Logs/qe_daemon.log` / `.err` (macOS) <br> `journalctl --user -u qe-daemon` (Linux) | stdout / stderr from launchd / systemd. |
| `~/Library/Application Support/qe_daemon/logs/<config-stem>-<pid>.log` (macOS) <br> `$XDG_STATE_HOME/qe_daemon/logs/...` (Linux) | Per-run daemon log. Written by dashboard-spawned **and** terminal-launched daemons — the daemon self-bootstraps a file sink unless stdout is already redirected to a regular file (so an explicit `> file` isn't double-written). |
| `~/Library/LaunchAgents/com.jiucheng.qedaemon.plist` (macOS) <br> `~/.config/systemd/user/qe-daemon.service` (Linux) | Service template, written by `qe_daemon install`. |
| `~/Documents/quant-strategy/*.qe` | Strategy configs. The `live(...)` wrapper is what makes one runnable by the daemon. |

## Writing a `live(...)` config

See [Daemon configuration and accounting](daemon-configuration.md) for this reference.

??? info "Moved section links"

    - <a id="multi-leg-cross-sectional-signalize_universe"></a> [Multi-leg + cross-sectional (signalize_universe)](daemon-configuration.md#multi-leg-cross-sectional-signalize_universe)

    - <a id="daily-resolution-strategies"></a> [Daily-resolution strategies](daemon-configuration.md#daily-resolution-strategies)

    - <a id="what-the-live-engine-reproduces-from-your-backtest"></a> [What the live engine reproduces from your backtest](daemon-configuration.md#what-the-live-engine-reproduces-from-your-backtest)

    - <a id="indicator-warmup-is-sized-from-your-config"></a> [Indicator warmup is sized from your config](daemon-configuration.md#indicator-warmup-is-sized-from-your-config)

    - <a id="picking-risk-limits"></a> [Picking risk limits](daemon-configuration.md#picking-risk-limits)

    - <a id="what-the-daily-loss-number-is-measured-against-epic-88"></a> [What the daily-loss number is measured against (EPIC-88)](daemon-configuration.md#what-the-daily-loss-number-is-measured-against-epic-88)

    - <a id="rebuilding-the-ledger-qe_daemon-book"></a> [Rebuilding the ledger — qe_daemon book](daemon-configuration.md#rebuilding-the-ledger-qe_daemon-book)

    - <a id="whose-shares-the-caps-count-qe_daemon-baseline"></a> [Whose shares the caps count — qe_daemon baseline](daemon-configuration.md#whose-shares-the-caps-count-qe_daemon-baseline)

    - <a id="the-order-and-it-is-not-negotiable"></a> [The order, and it is not negotiable](daemon-configuration.md#the-order-and-it-is-not-negotiable)

    - <a id="reading-the-table"></a> [Reading the table](daemon-configuration.md#reading-the-table)

    - <a id="what-each-refusal-means"></a> [What each refusal means](daemon-configuration.md#what-each-refusal-means)

    - <a id="what-the-cut-over-leaves-behind"></a> [What the cut-over leaves behind](daemon-configuration.md#what-the-cut-over-leaves-behind)

    - <a id="reverting"></a> [Reverting](daemon-configuration.md#reverting)

    - <a id="when-baseline_stale-fires"></a> [When baseline_stale fires](daemon-configuration.md#when-baseline_stale-fires)

## Upgrading an existing deployment

Read this **before** you restart a running daemon onto a newer build.
Three 0.4.0 changes can turn a config that started yesterday into one
that refuses today. All three are refusals, not silent behaviour
changes, and all three exit before a socket to the venue is opened.

The whole check is one command, and it contacts no broker:

```bash
qe_daemon validate path/to/live_config.qe   # exit 0 = deployable
```

Run it against the *currently deployed* config, on the *new* binary,
while the old daemon is still up. If it exits 0, restart normally.

### Changing a config without restarting

A new binary needs a restart. A **config edit** may not.

```bash
qe_daemon reload-check path/to/live_config.qe   # what would change, and could it be applied
qe_daemon reload-apply path/to/live_config.qe   # commit the subset that is safe right now
```

`reload-check` binds the candidate, diffs it against what the daemon is
running and prints one line per field with a category. It commits
nothing, and it is on the read-only allow-list, so it works through a
forwarded socket ([Watching a daemon on another
machine](#watching-a-daemon-on-another-machine)). Exit 0 means the
changes are appliable; exit 3 means they need a flat book or a restart,
so a deploy script can branch without parsing the output.

| category | meaning | example |
|---|---|---|
| `hot` | applied to a running daemon | the four `risk_max_*` caps |
| `flat_only` | needs a flat book; **not yet implemented**, so refused | the universe, the factor, `capital` |
| `restart_only` | never appliable | `broker`, `journal_dir`, the `ibkr` endpoint |

`reload-apply` refuses twice: once if **any** change is not `hot`, and
once if **any** slice is still pending. The second is the same
condition that makes `stop + start` unsafe — a queued slice is an order
the daemon has already committed to, and moving a risk cap underneath
it measures it against a limit nobody chose for it. Clear the window or
`cancel-all` first.

Two things it does not do. It does **not** run the deployability gates
above — `validate` owns those, and `reload-check` says so in its own
output. And it refuses a file whose only change is a comment: the
differ compares source bytes as a backstop, cannot tell a comment from
a field it does not model, and refuses rather than reporting a
misleading "no changes".

Every applied reload writes a `config_reload` row to the journal, so a
hot reload leaves the audit trail a restart would have left.

### 1. `rolling_vol` on a bare price is now fatal (BREAKING)

`rolling_vol(close, 20)` measures dollar-price dispersion. Live validation
refuses the bare-price form; `qe_run` still accepts it for reproduction.

- For return volatility, use `realized_vol(close, 20)` and validate the
  resulting strategy again: this changes the factor.
- For intentional dollar dispersion, use
  `rolling_vol(price_level(close), 20)`: the arithmetic is unchanged.
- `--allow-price-level-vol` is a temporary override that logs an error
  on every startup.

See the [0.4.0 migration](release-history.md#040-qe_daemon-refuses-a-price-level-volatility-factor)
for the full explanation. Validate the edited configuration before restarting.

### 2. A short warmup now refuses instead of trading on NaN

The daemon used to request a fixed `"60 D"` (~41 trading bars) for
every config ever written and start regardless of what came back. It
now derives the requirement from the config — every leg's entry, exit
and sizing expression — and **refuses when any symbol delivers fewer
bars than the strategy needs**.

A config that started yesterday and refuses today was not working
yesterday: it was evaluating its factor on NaN, and NaN sorts *last* in
the cross-sectional pre-pass, so a long-only `top_k` selection was
quietly picking from whichever symbols happened to be warm. The failure
is now visible where it used to be silent.

The most likely trigger is a recently-listed name in your universe: if
it has 90 bars of history at the venue, no phrasing of the request
produces 252. Either shorten the longest lookback or drop the name.
`qe_daemon validate` prints the requirement, the driving indicator and
the leg it came from without contacting a broker.

### 3. Reconciliation drift now blocks submissions

Drift that previously logged at WARN and kept trading now stops all new
submissions — **entries and exits** — until an operator acknowledges it
with `qe_daemon reconcile-ack`. If you have a supervisor script that
treats "daemon is up and healthy" as "daemon will trade", it needs to
check `qe_daemon reconcile` (exit 3 while blocked) or the
`reconcile_blocked` field on `status`. Details in
[When something goes wrong](#when-something-goes-wrong) below.

### Rolling back

Nothing in these three changes touches the journal format in a way an
older binary cannot read, so downgrading is possible. The new
`session_anchor` / `strategy_book` / reconcile-block rows are journal
notices, which an older daemon replays as unrecognised notices and
ignores — you lose the restored anchor and any outstanding block, which
is exactly the pre-0.4.0 behaviour. Read from the replay code, not
exercised against an older binary in this environment: verify on paper
before rolling back a live deployment.

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
   - `historical warmup = N bars across M symbols (min K per
     symbol, needed R)` — R is derived from the config's
     `min_history`, not a fixed 60 D window
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
   Prints a grouped status report — SAFETY, EXPOSURE, ORDERS,
   RECONCILE — each led by a verdict line. That form is for reading,
   not for parsing: it is prose and it changes.

   **`qe_daemon status --json` prints the raw payload**, and that is
   the form to parse in a script — it is also what the dashboard's
   DAEMON chip mirrors. Two places below tell you to script against
   `status` and read `reconcile_blocked`; they mean `--json`. Piping
   the human report into a JSON parser is not a hypothetical mistake:
   `scripts/qe-daemon-supervisor.sh:370` did exactly that and paged an
   operator about a restart that had worked.

6. **Attach the dashboard** (any time after step 5). The top-bar
   DAEMON chip flips green within ~2 s. F6 TRADE's safety panel
   reads:

   ```
   SAFETY · kill: armed · reconcile: clean · session: PAPER
          · daemon: attached · venue: connected
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
   - **`venue:`** — the daemon's view of its broker link
     (EPIC-70). `connected` (green) is healthy.
     `PAUSED · watchdog · N missed poll(s)` (amber; N counts failed polls, not seconds) means the disconnect watchdog tripped
     the soft-pause — new orders reject for the duration, open
     orders untouched. `TRIPPED (reason)` (red) means the
     watchdog escalated to the hard trip; operator must restart.

7. **Tail events** while you watch the first session:
   ```
   qe_daemon tail events
   ```

## Going live — additional gates

`broker = "ibkr-live"` flips two things:

* The live broker refuses to construct without the env gate
  `IBKR_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING`
  (`src/net/ibkr_broker.cpp:187`; the token is
  `include/qe/net/broker.hpp:518`). It is the exact string — not
  `1`, and not `QE_LIVE_TRADING`, both of which this page named
  until 2026-08-11 and neither of which exists anywhere in the
  codebase.
* The dashboard's broker chip paints red `LIVE`. The dashboard's
  kill-switch is one keystroke away (`Cmd-Shift-X`). There is no
  confirmation modal — the only feedback is a red `· KILL` chip in
  the top bar, whose tooltip prints the trip reason
  (`apps/dashboard/gui_chrome.cpp:101-108`).

> Practice tripping the kill-switch in paper before you go live.
> You should know what the dashboard looks like in the OFF state
> before you ever need to use it under pressure.
>
> And practice the thing the kill-switch does *not* do: clearing a
> working order. Place a far-from-market limit order on paper, trip
> the switch, watch the order sit there untouched, then clear it —
> with `qe_daemon cancel-all --yes`, and again by hand in TWS. That
> is the muscle memory you actually need, and doing it once in
> paper is how you learn what the report looks like when it comes
> back partial — see
> [Emergency stop](#emergency-stop-orders-are-working-and-i-need-them-gone).

The promotion procedure:

1. Paper-deploy with the same `.qe` for **at least 5 sessions**
   with no manual interventions or unexpected events in the tail.
2. Flatten any positions at IBKR-paper.
3. Edit the config: `broker = "ibkr-live"`. Re-run risk-limit
   sanity check.
4. `qe_daemon uninstall` to drop the paper service.
5. `IBKR_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING qe_daemon install <live_config.qe>`.
   The variable must be present in the environment the service
   actually runs under, not just the shell you typed `install` in.
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
| See which book the risk caps are measuring | `qe_daemon status` → EXPOSURE block, or `qe_daemon baseline` |
| Declare which of the account's shares are yours (once, and again after you trade by hand) | `qe_daemon baseline propose` → check the table against TWS → `qe_daemon baseline adopt --token=<t> --yes` → restart. Read [Whose shares the caps count](daemon-configuration.md#whose-shares-the-caps-count-qe_daemon-baseline) first; run it before the open. |
| Stop the daemon submitting anything new, from CLI | `qe_daemon kill --yes` (EPIC-88 T88.11). **One-way** — nothing clears it but a restart. Blocks new submissions; queued staged slices are **preserved** (a restart re-evaluates them); working orders are untouched. |
| Hold submissions *reversibly* | `qe_daemon pause` / `qe_daemon resume`. Same gate the disconnect watchdog uses for a soft pause. Use this, not `kill`, when you expect to carry on today. |
| Trip the *dashboard's* kill-switch | F6 TRADE → "Trip kill-switch", or `Cmd-Shift-X`. Note this trips the **dashboard's own** switch, not the daemon's. To kill the daemon from the dashboard use F6 SAFETY → DAEMON KILL (arm-then-confirm, EPIC-88 T88.10). |
| **Cancel orders that are already working** | `qe_daemon cancel-all --yes` (EPIC-88 T88.1), or TWS / IB Gateway / Client Portal by hand. See [Emergency stop](#emergency-stop-orders-are-working-and-i-need-them-gone) — read the report it prints, it is allowed to come back partial. |
| Uninstall the LaunchAgent / unit | `qe_daemon uninstall` |

> **`qe_daemon kill` exists as of EPIC-88 T88.11.** It did not
> before: the subcommand dispatch knew `start` / `run` / `status` /
> `stop` / `tail` / `install` / `uninstall` / `validate` / `smoke` /
> `backfill`, and everything else **fell through to `cmd_start`**, so
> `qe_daemon kill panic` tried to start a *second daemon* using the
> string `"kill"` as a config path (defect KS-5). The fall-through is
> now narrowed: a first token with no `/` and no `.qe` suffix is an
> error (exit 2) that lists the verbs, instead of a guess. The legacy
> positional form — `qe_daemon strat.qe`, `qe_daemon ./cfgs/x.qe` —
> still works.

### Emergency stop — orders are working and I need them gone

Three commands, in this order. The order is the whole point; the
rest of this section is why.

```bash
qe_daemon kill --yes         # 1. nothing new goes out
qe_daemon cancel-all --yes   # 2. clear what is already working
qe_daemon stop               # 3. only once step 2 came back clean
```

1. **Kill.** Trips the daemon's kill-switch: new submissions are
   blocked. Queued staged slices are **preserved** — they cannot
   fire while killed, and a restart re-evaluates them, firing those
   still inside their window and retiring the rest as
   `window_miss`. It cancels nothing at the venue — that is
   decision D1 and it is deliberate. Without `--yes` you get a confirmation prompt; in a
   script `--yes` is mandatory, because a prompt written into a
   pipe hangs. **One-way** — only a restart clears it.
2. **Cancel-all.** Sends the daemon's `cancel_all` control verb,
   which cancels every order the broker currently lists as
   working. Read what it prints:

   ```
   found:        3 working order(s) at the venue
   acknowledged: 2
   failed:       1
     NOT cancelled: 000e1f4c.a1 AAPL — order not found
     Those orders may still be working. Cancel them by hand in TWS / IB Gateway /
     Client Portal before stopping the daemon.
   ```

   Exit 0 means the venue listed the book *and* acknowledged every
   cancel in it — including the honest zero, "nothing was
   working". **Exit 3 means the book may not be clear**, and there
   are two ways to get it: a cancel the venue refused (above), or
   a link that could not be listed at all:

   ```
   FAILED: could not list open orders: ibkr_connection: not connected
     NOTHING was cancelled, and the number of orders working at the venue is
     UNKNOWN — the broker link is what we would have needed to find out.
     Cancel by hand in TWS / IB Gateway / Client Portal.
   ```

   That second shape is the one to internalise. A dead link is
   exactly when a summary saying "0 orders cancelled" would be
   most reassuring and least true, so the report refuses to print
   a count it does not have. If you see it, the daemon cannot help
   you and TWS is the answer.

   Every run lands an `operator_cancel_all` row in the journal
   carrying the counts and **each failure's reason**, plus the
   usual `CancelAttempt` / `CancelAccepted` / `CancelFailed`
   triples in the order log. You can reconstruct the whole attempt
   afterwards.

3. **Stop**, once step 2 reports clean — F6 TRADE → "Stop daemon",
   or `qe_daemon stop`.

**Do not reverse 2 and 3.** Between stopping the daemon and
cancelling, a working order can still fill. The venue does not know
or care that the process that sent it has exited, and with the
daemon dead there is nothing left to book that fill, write it to
the journal, or react to the position it just opened. You would
discover it at the next session start, from the broker statement.

**Do not skip step 1** either. `cancel-all` does not trip anything,
so on a live, un-killed daemon it clears the book and the strategy
refills it at the next evaluation. The report says so when it
happens:

```
WARNING: the daemon's kill-switch is NOT tripped. It can submit new orders, and
         a strategy that still wants a position will simply place them again.
```

Two things `cancel-all` deliberately does not do:

- **It is not reachable from a kill-switch trip.** A trip blocks
  submissions; cancelling is a separate act with its own
  confirmation. `tests/test_daemon_control_handler.cpp`
  (`"cancel_all is NOT reachable from a kill-switch trip"`) asserts
  a trip issues zero cancels, so the wiring cannot drift back.
- **It does not touch staged slices** — those are queued inside
  the daemon, not orders at the venue. When any are pending the
  report names the count and points at `kill`, which drops them.

`cancel-all` cancels *every* working order on the account, not
only the ones this daemon placed. If you hand-trade the same IBKR
account, that is your order too.

See [Live-trading safety](live-trading-safety.md#layer-3-kill-switch)
for exactly what a trip does and does not do, with file:line.

### A daemon kill-switch trip notifies you (EPIC-88 T88.8)

A trip is the single most important thing this system can tell you,
and until T88.8 it told you nothing you would find in time: an
`spdlog::error` line and a `kill_switch_trip` journal row, both
perfectly durable and both discovered only by somebody already
looking. The July paper deployment latched at 10:04 and was noticed
at 16:00 by someone wondering why nothing had traded.

Every trip — the disconnect watchdog, the daily-loss breaker, the
daemon's own trip path, and the control socket's `kill` verb — now
also raises a **desktop notification**:

| Where the daemon runs | Channel |
|---|---|
| macOS (LaunchAgent, so inside your Aqua session) | Notification Center, via `/usr/bin/osascript` |
| Linux with a session bus | `notify-send`, urgency critical |
| Headless (server, LaunchDaemon, no session bus) | `syslog(LOG_CRIT)` |

The same three channels `scripts/verify-open-check.sh` and
`scripts/qe-daemon-supervisor.sh` already use — no new dependency,
and nothing in `qe_daemon` speaks HTTP. If you want a trip pushed off
the box (Slack, phone), point `QE_SUPERVISOR_WEBHOOK` at it in the
supervisor script; that lives out of process on purpose.

The notification repeats the D1 rule verbatim, because for most
people it is the only sentence they will read:

```
qe_daemon KILL SWITCH TRIPPED
reason: daily-loss-kill
New orders are BLOCKED. Orders already working at the venue were NOT
cancelled - cancel them by hand in TWS.
```

Two properties worth trusting:

- **The journal row is written first.** The notification is
  best-effort and comes second, so a process that dies between the
  two loses the toast, never the audit record. A notification channel
  that fails, or throws, cannot delay or undo the trip.
- **It does not block.** The alert is double-forked and exec'd, so
  the thread that tripped — the watchdog poll, or the socket
  answering `kill` — returns in microseconds rather than waiting on a
  toast being drawn.

To silence the desktop channel, set `QE_OPERATOR_ALERT=off` (also
`0`, `no`, `none`) in the daemon's environment. The journal row and
the error log line are written regardless; nothing turns those off.
**Check `qe_daemon.out` at start-up for the line that says which
channel is armed** —

```
[info] daemon: operator alert: a kill-switch trip will be pushed via osascript.
[warning] daemon: operator alert: DISABLED by QE_OPERATOR_ALERT — ...
```

— because a variable left set in a shell profile or a plist is
invisible until the trip that needed it.

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

### The weekly deploy check

`scripts/verify-open-check.sh` answers "is the deployment actually
alive and actually trading?" — daemon reachable, kill-switch not
latched, not paused, orders being submitted *and* filled, no stuck
slices. Install it where the launchd job expects it:

```bash
scripts/install-open-check.sh              # install / refresh
scripts/install-open-check.sh --verify     # report drift, change nothing
scripts/install-open-check.sh --diff       # show installed-vs-repo diff
```

Exit codes: `0` healthy, `3` daemon reachable but something is wrong,
`4` no daemon running, `5` the check itself could not run. On any
non-zero exit it writes `checks/ALERT` **and raises a macOS
notification**; a clean run removes `ALERT`.

> The notification is the point. The previous version of this check
> ran every Monday for seven weeks while the paper deployment was
> dead and told nobody — it had no `set -e`, its last command was
> `ln`, so it exited 0 every time and launchd saw seven clean runs.
> The reports it wrote were accurate and unread. **A check that
> cannot wake you is not a check.** If you re-work it, keep the
> notification path and verify it fires by running with the daemon
> stopped.

#### Install it with the installer, never by hand

launchd does not run the file in the repo. It runs a **copy** at
`~/Library/Application Support/qe_daemon/checks/verify_open.sh`, and
those two silently diverged for weeks: the installed copy predated
EPIC-83 T83.23, so the weekly check could not report the risk
projection triple at all. A whole class of fault was invisible to the
one job whose entire purpose is making faults visible, and nobody
noticed — a stale check still runs, still writes a report, still
exits 0.

`scripts/install-open-check.sh` copies from the repo, verifies the
copy's SHA-256, and writes a manifest next to it recording what was
installed, from where, and from which commit. Section 0b of the check
reads that manifest back **on every run** and raises a PROBLEM (so:
`ALERT`, notification, exit 3) when

- the installed copy no longer matches the repo — *stale install*,
- the installed copy was edited in place — the repo has never seen
  those lines, so harvest them before reinstalling,
- there is no manifest at all — provenance unknown, which is what a
  hand `install -m 755` leaves behind.

The installer refuses to overwrite an installed copy containing lines
the repo lacks, prints the diff, and tells you to bring them back into
the repo first; `--force` overrides once you have looked. Hand-editing
the manifest to silence a drift alert re-creates the exact defect the
manifest exists to prevent.

Re-run `scripts/install-open-check.sh` after **every** edit to
`scripts/verify-open-check.sh`. Nothing else propagates it.
`tests/scripts/test_open_check_install.sh` covers the install, drift
and refusal paths against a throwaway `$HOME`.

#### `STANDING SUBMIT FAULT` — how section 4 decides

Section 4 reads every `order_rejected` / `order_submit_failed` row in
every journal segment and asks one question **per symbol**: what was
the last word? If the newest submission-related row for that symbol is
a refusal, nothing has reached the venue for it since, and the fault is
reported as **standing** — at any age. If something the venue accepted
came after it (an `order_fill`, or a `slice_fired` carrying a real
`broker_id`), the fault has been retested and passed, and the row drops
to the `superseded:` line as history.

Clearing one is therefore not a matter of waiting. **Fix the cause, then
let the strategy trade that symbol once.** The successful submission is
the evidence, and it is the only evidence there is.

> Why not a 14-day window, which is the obvious repair for "one June
> rejection alarms every Monday forever"? Because the deployed strategy
> rebalances every ~21 bars. Its newest refusal can age past two weeks
> before the next close is even evaluated, so "nothing in the window"
> would mean both *healthy and idle* and *broken since before the
> window* — and a standing configuration fault (tws_code 201, missing
> permissions, a name the account cannot short) would go silent exactly
> between rebalances. Widening the window does not fix that; wall-clock
> age is simply not evidence about a cadence-driven strategy.

The residual case is a symbol that is refused and then leaves the
universe: nothing will ever supersede it, so it alarms until the
segment holding it is archived out of `state/`. Read the alarm on its
own terms — the account still cannot trade that name — and note that
archiving a segment rewinds `EvalRoundIndex::count_for` and therefore
the rebalance phase, so it is not a routine operation.

### Logs and the journal

Two different files, two different rules. Getting them confused
costs you the exact evidence you need after an incident.

> **Never edit `events.jsonl` by hand.** The restart-replay
> reconciler reads it; an out-of-shape line will fail loudly but
> a *plausible* edit can silently desync state.

#### Never `rm` a log file the daemon is holding open

On Unix, deleting a file the daemon has open does **not** free the
disk and does **not** give you a fresh log. The daemon keeps writing
to an unlinked inode that no path points at any more; every line
from that moment until the process exits is unreachable, and the
space isn't reclaimed until it exits either. Same for `rm -rf` on the
whole log *directory* — a new file is not created, because the sink
was opened once at startup.

This is not hypothetical. On **2026-06-20 02:30:49** the
`~/Library/Application Support/qe_daemon/logs/` directory was
removed while pid 14953 was running. That daemon had been up since
06-18 and stayed up afterwards; the entire process lifetime of
logs — including the 06-19 IB socket death and the watchdog
kill-switch trip that cancelled all ten queued slices (staged,
not-yet-submitted — nothing at the venue was touched) — was
unrecoverable. The post-mortem had to be reconstructed from the
journal alone.

Truncate in place instead. The write offset resets, the inode
survives, the daemon keeps logging:

```bash
: > ~/Library/Application\ Support/qe_daemon/logs/<stem>-<pid>.log
```

To reclaim space across old runs, delete only logs whose pid is
**not** running:

```bash
cd ~/Library/Application\ Support/qe_daemon/logs
for f in *-*.log; do
  pid="${f##*-}"; pid="${pid%.log}"
  kill -0 "$pid" 2>/dev/null || rm -- "$f"
done
```

#### launchd: a log path under `~/Documents` kills the agent silently

Before the section below, the harder one. **Never point
`StandardOutPath` or `StandardErrorPath` at a file inside
`~/Documents`, `~/Desktop` or `~/Downloads`.** Those are TCC-protected.
A launchd agent that names one fails to spawn with `EX_CONFIG` (78)
unless launchd itself has Full Disk Access, and it fails with no
diagnostic anywhere — the file it could not open is the file the error
would have gone to.

This bites the *supervisor* specifically, because its natural home is
the workspace and the workspace is usually in `~/Documents`. Observed
2026-08-01: `ca.jiucheng.qe-daemon-supervisor` had been dead ~8 hours
while `qe-daemon-supervisor.sh` ran fine by hand. The agent that exists
to notice a dead daemon was itself dead, and by construction could not
say so.

How to see it:

```bash
launchctl list | grep qe-daemon
#  -   78   ca.jiucheng.qe-daemon-supervisor     <-- spawn failed
#  -    0   ca.jiucheng.qe-daemon-supervisor     <-- ran, exited clean
```

The middle column is the last exit status. A persistent `78` with no
new lines in `supervisor.log` means the job never started; check the
log paths before you debug the script. Keep them under
`~/Library/Application Support/qe_daemon/supervisor/`, and create that
directory first — launchd will not `mkdir` a missing parent, and that
failure presents identically.

**Add `launchctl list | grep qe-daemon` to whatever you check weekly.**
Status 78 is invisible to every other signal in this runbook: the
daemon looks healthy, because it is — nothing is watching it.

#### launchd: `StandardOutPath` silences the per-config log

`install_log_file_sink()` skips its own file sink when stdout is
**already a regular file**, on the assumption that an outer wrapper
is capturing output and a second sink would double every line. That
is right for the dashboard's double-fork and for an explicit
`> file`, and it is also true of launchd: the generated plist sets

```xml
<key>StandardOutPath</key>
<string>{{LOG_DIR}}/qe_daemon.log</string>
<key>StandardErrorPath</key>
<string>{{LOG_DIR}}/qe_daemon.err</string>
```

so a launchd-started daemon writes to those two files and
`logs/<stem>-<pid>.log` **is never created**. Nothing is lost, but
the file you reach for by habit isn't there.

Consequences if you adopt `qe_daemon install`:

- Read `~/Library/Logs/qe_daemon.log` / `.err`, not
  `logs/<stem>-<pid>.log`.
- Any glob, log-tail script or alerting rule keyed on
  `logs/*-<pid>.log` must be updated in the same change — a script
  that finds nothing tends to report "healthy".
- The two launchd files are per-*label*, not per-run: every restart
  appends to the same pair, with no pid in the name. Rotate them
  yourself (`: > qe_daemon.log`, per the rule above — the daemon
  holds them open too).
- systemd is different again: output goes to the journal, read it
  with `journalctl --user -u qe-daemon`.

#### systemd: the supervisor has to be installed separately, and it wasn't

The supervisor is the only thing that notices a daemon which is simply
**gone**. Every liveness signal the daemon itself publishes —
`last_tick_ns`, `tick_stalls`, the missed-eval worker — is in-process,
and a process cannot report its own death.

The unit files have existed as long as the launchd plist has. What did
not exist was a way to *install* them: the procedure was a `sed`
pipeline written in a comment at the top of the unit file. On the Linux
host it had never been run, the deploy script never mentioned
supervision, and nothing anywhere reported that the observer was
absent.

What that cost, measured from the journal: the paper daemon stopped on
**2026-08-24 and the journal was silent for 21.7 days**. The first row
after the hole is an `eval_missed`. No alarm fired anywhere, because
there was no alarm running.

The lesson generalises past this one host. An install procedure that
lives in a comment is one that does not run, and a component whose
absence is silent will eventually be absent.

Install it:

```bash
scripts/install-supervisor-systemd.sh \
    --config  backtests/<...>/deploy.qe \
    --workdir ~/quant-strategy \
    --ntfy    https://ntfy.sh/<your-topic>
```

It renders a user service + a 10-minute timer, turns on lingering
(without it the supervisor dies with your SSH session, which is exactly
when you stop being able to notice), and then **asks systemd** whether
the timer is really active rather than reporting success on faith.

Check it the way you check `launchctl list` on macOS:

```bash
systemctl --user list-timers qe-daemon-supervisor.timer
journalctl --user -u qe-daemon-supervisor -n 20
```

**On the notification channel.** This is the part that decides whether
any of it matters. With no channel configured, `notify()` falls back to
`notify-send` (needs a desktop session you are sitting at) or
`logger(1)` → journald, which nobody reads. That is the same alarm in
the same empty room, and it is the configuration the 21.7-day outage
ran in. Set `QE_SUPERVISOR_NTFY` or `QE_SUPERVISOR_WEBHOOK`; the
installer exits non-zero and says so if neither is present.

Both URLs are credentials — an ntfy topic URL lets anyone holding it
read and post. They live in `notify.env` at mode 0600 under the
supervisor's state dir, passed to the unit via `EnvironmentFile` so
they never appear in `/proc/<pid>/cmdline`.

The unit deliberately does **not** pass `--start-if-absent`. "Absent"
cannot be told apart from "an operator stopped it", and the standing
rule here is that a daemon holding orders is never restarted by
automation. Telling a human is the whole job.

#### Journal rotation

The active segment is `events.jsonl`. When the next row would push
it past 64 MiB it is renamed `events-<YYYYMMDDThhmmssZ>.jsonl` and a
fresh empty `events.jsonl` takes over. `replay()` reads **every**
segment in order, so rotation is invisible to restart recovery.

Retention is unbounded on purpose. Rotated segments are never
deleted, truncated or overwritten — the rebalance phase is seeded by
counting distinct evaluated closes across the whole journal, so
dropping the oldest segment would silently rewind the phase and
re-fire an already-executed rebalance. At the observed rate
(~267 KB per three weeks of one daily deployment) 64 MiB is a
safety valve for a pathological error loop, not a routine event.

If disk ever matters, **move segments out of the journal directory**
rather than deleting them, and understand that you are shortening
the phase history when you do.

## When something goes wrong

### Daemon says it can't reach the broker

The disconnect watchdog (EPIC-70) is a 3-state machine: **Healthy
→ Paused → Tripped**. What you see depends on how long the
broker has been offline:

| Wall-clock offline | Watchdog state | F6 `venue:` badge | Effect |
|---|---|---|---|
| ≤ 3 s | Healthy | `connected` | Normal — TCP heartbeat blip absorbed |
| 3 – 30 s | **Paused** | `PAUSED · watchdog · N missed poll(s)` (amber) | New orders reject (`PreTradeRisk` gate). Open orders untouched. Auto-resumes on reconnect. |
| > 30 s | **Tripped** | `TRIPPED (ibkr_disconnect)` (red) | KillSwitch tripped; pending staged slices **preserved** — a restart re-evaluates them. **Open orders at the venue are NOT cancelled** — a cancel needs the broker link, which is the thing that just died. The journal gets the count of orders left working, or `unknown` when even that could not be read. Cancel them by hand in TWS. Operator must restart. |

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

- `daily-loss kill` tripped → expected; restart only after you
  investigate the equity drop. Note the trip **does not by itself
  stop the process** — the daemon stays up, keeps consuming bars,
  and keeps booking fills; it just submits nothing new. If the
  process is actually gone, something else killed it. Since EPIC-88
  T88.4 the trip can come from either the continuous evaluation (two
  consecutive 30 s marks past the limit, no order involved) or the
  submit-time backstop; the `daily_loss_breach` journal rows carry
  the P&L and the breach streak that decided it.
- Reconcile-vs-broker drift at startup → broker positions don't
  match the journal. **Don't auto-fix.** The daemon refuses to
  trade and points you at a drift report. Flatten at the broker,
  or reconcile by hand and restart.

  There is **no `--reset-journal` flag** — an earlier draft of this
  runbook promised one and it never existed in any source file.
  Erasing the record at the exact moment the record and the broker
  disagree is the worst possible time to do it, and it is not
  needed: startup re-seeds positions from `list_positions()`
  (broker truth wins over replay), so a drift stop is a
  *disagreement to investigate*, not state to delete. If you truly
  need a clean slate, archive rather than delete — with the daemon
  stopped, `mv events.jsonl events-$(date -u +%Y%m%dT%H%M%SZ).jsonl`
  in the journal dir. That is exactly the rotated-segment name, so
  the history stays replayable instead of vanishing.
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
or grep the journal. The reasons:

- `per-symbol position cap`
- `gross exposure cap`
- `daily loss kill` (also tripped the switch)
- `orders-per-minute throttle`
- `kill-switch tripped`
- `no mark price for symbol` — a notional cap is armed and some
  position it must value has no usable mark, so the gate fails closed
  rather than valuing that holding at $0. The message names the
  symbol, which is not always the one you ordered
- `long-only violation` — the SELL would have left the account SHORT.
  Not a limit you can widen: this engine cannot hold a short position,
  so the order was wrong, not the number. The usual cause is an exit
  sized off a book that overstates the holding; `qe_daemon status` and
  the broker's own position for that symbol are where to look
- `strategy short inversion` — the SELL would have left the STRATEGY
  short while the account stayed long, i.e. it sold shares that after
  the order only your declared baseline can back. Same flag as the
  previous one and just as unwidenable, but a different book and a
  different fix: this one is almost always a **stale baseline**. Run
  `qe_daemon baseline` and compare the declared external quantity for
  that symbol against what you actually hold by hand — if you have
  hand-sold since you adopted it, re-adopt. It cannot fire at all
  unless a baseline is established

**Don't widen a limit "for one trade".** Either the limit was
miscalibrated (raise it permanently after a deliberate review)
or the strategy is misbehaving (fix the strategy).

**On either of the first two, check the scope before you touch the
number.** Run `qe_daemon status` and read the EXPOSURE block. `⚠
ACCOUNT` means both caps are measuring the whole IBKR account, so a
buy can be refused over stock the strategy never bought and no number
you can type there is the right one — raising the cap just moves the
same wrong measurement further out. [Whose shares the caps
count](daemon-configuration.md#whose-shares-the-caps-count-qe_daemon-baseline) is the fix.
`⛔ CLAMPED` means a baseline *is* declared and no longer fits the
account, which is the same conversation one step further along.

## Watching a daemon on another machine

The daemon does not have to run where the dashboard runs. Once it
lives on a deployment host, the dashboard reaches it through an
SSH-forwarded socket — **read-only**, enforced by the daemon.

### Why read-only is enforced on the far side

The control socket has no authentication of its own. Its whole
security model is filesystem ownership, and reads share the socket
with `kill`, `cancel_all`, `stop` and `reconcile_ack`. So forwarding
the *ordinary* socket hands out full control of a live trading
process to anything that can reach the forwarded endpoint.

The daemon therefore binds a second socket that serves observation
only:

```
/run/user/<uid>/qe_daemon.sock       full control  — local to the host
/run/user/<uid>/qe_daemon-ro.sock    read-only     — safe to forward
```

The read-only endpoint permits `status`, `positions`, `orders`,
`equity`, `eval_history`, `eval_alerts`, `scheduled_orders`,
`log_tail`, `reconcile_status`, `book`, `baseline` and
`reload_check` — twelve verbs, enumerated by `is_read_only_verb` in
`apps/daemon/read_only_control_handler.hpp`. (Cited by function name
rather than line range on purpose: the range this used to carry went
stale the first time a verb was inserted, and pointed at an
`#include`.) Everything else is refused
with `ReadOnly`, whoever asks. It is an allow-list: a verb added to the
daemon later is refused until someone opts it in deliberately.

`reconcile_ack` counts as a mutation. It touches nothing at the
venue, but it releases the submission block — so acknowledging drift
requires being on the host. That is the one restriction you will
actually feel.

`book` is a read, including its `rebuild` dry run: that form replays
the journal into a scratch ledger and hands back a token, and writes
nothing. The commit is a separate verb, `book_rebuild`, which is not
on the list — so a rebuild can be *planned* from the dashboard's host
and only *applied* from the daemon's.

`baseline` splits the same way, and this matters more than the book
one because the EPIC-90 cut-over is the procedure most likely to be
attempted from an observing host. The read and `baseline propose` —
the dry run that computes the split, prints the table and hands back
a token — are both served read-only. The three mutators
(`baseline_adopt`, `baseline_set`, `baseline_disable`) are separate
verbs and are **not** on the list. So a split can be proposed over
the forward, read against TWS at leisure, and adopted only from the
daemon's own host.

### Establishing the forward

```bash
ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 \
    -L /tmp/qe-ro.sock:/run/user/1002/qe_daemon-ro.sock user@host &
```

`ExitOnForwardFailure` matters: without it, ssh connects and silently
fails to bind the socket. That used to leave a dashboard saying "no
daemon" for a reason nowhere on screen; it is now visible —
a configured socket with nothing at the path paints the badge amber
rather than blank, because "never had a daemon" and "had one and it
went away" are different problems and only one of them is yours to
fix. Keep the option anyway: an amber badge you have to interpret is
worse than a forward that simply works.

Find `<uid>` with `id -u` on the host. Remove a stale
`/tmp/qe-ro.sock` first — ssh will not overwrite one.

**This is an ordinary process, not a service.** Laptop sleep, a
network change or a reboot ends it. When it dies the dashboard shows
the daemon as gone **while the daemon keeps trading** — the
observation channel and the trading channel are deliberately
independent, which is the point of moving the daemon off a laptop,
but it is worth knowing before the first time you see it.

### Pointing the dashboard at it

Settings → Trading → **Daemon control socket**, or `daemon_socket` in
`config.json`:

```json
"daemon_socket": "/tmp/qe-ro.sock"
```

Blank means the platform default, i.e. a daemon on this machine.

Edit the file only while the dashboard is closed — it rewrites its
config on every change and will overwrite you.

The CLI takes the same path through the environment:

```bash
QE_DAEMON_SOCKET=/tmp/qe-ro.sock qe_daemon status
```

There is no `--host` flag anywhere, deliberately. The transport is
SSH's job; a host flag would mean inventing authentication for a
socket that has none.

### Confirming it worked

The top bar shows `@<host>` next to the DAEMON badge whenever the
daemon reports a hostname that is not this machine's, and `RO` when
the attachment is read-only. F6 TRADE adds a banner above the action
row, and disables the controls that would change the daemon —
disabled and visible, not hidden, so that an operator hunting for the
kill switch during an incident finds it and reads why it is greyed
out.

Both markers come from the daemon's `status` reply, not from the
socket path. The dashboard does not guess whether it can act.

A quick check from the command line:

```bash
QE_DAEMON_SOCKET=/tmp/qe-ro.sock qe_daemon status   # works
QE_DAEMON_SOCKET=/tmp/qe-ro.sock qe_daemon pause    # refused
```

If `status` reports the socket path you did *not* ask for, the
binary predates `QE_DAEMON_SOCKET` — check which `qe_daemon` is on
your PATH.

## Driving the daemon from Python

Use the [Python daemon client](python.md#1-the-daemon-client) for
examples, read/control verbs and typed errors. It uses only the Python
standard library and connects through the same control socket as the CLI.

To re-pin the client against a running daemon after changing either
side — read verbs only, safe against a live book:

```bash
ssh <host> 'cd ~/quant-strategy && python3 - --socket /run/user/1002/qe_daemon.sock' \
    < scripts/per240-pin-wire-format.py
```

## What ships, what doesn't

The daemon runtime supports **`ibkr-paper` and `ibkr-live`**. Alpaca
configurations parse but exit 4 when starting the daemon.

- **Kill is one-way:** restart the daemon to clear it; `unkill` is refused.
  A kill blocks submissions but does not cancel working venue orders or
  discard pending staged slices. Use explicit `cancel-all` or the broker UI
  for working orders; see [Emergency stop](#emergency-stop-orders-are-working-and-i-need-them-gone).
- **Per-order cancellation:** the daemon has `cancel_all`, but no
  `cancel_order`. F6's per-order Cancel stays disabled when attached;
  use the broker UI to cancel one order.
- **Restart recovery:** pending slices are restored, dispatched only inside
  their windows, and retired as `window_miss` if late. See
  [Staged entry / exit](staged-entry.md#restart-recovery).
- **Warmup and reconciliation:** startup needs enough history for every
  symbol; reconciliation can block entries and exits. See
  [current limitations](safety-and-limitations.md).

Run the [daemon smoke checklist](qe-daemon-smoke.md) after first install
and before live promotion. Historical paper verification is not evidence
that a new deployment passed those checks.

[live-trading-safety.md]: live-trading-safety.md
