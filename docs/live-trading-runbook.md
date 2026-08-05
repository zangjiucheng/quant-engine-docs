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
| `~/Library/Application Support/qe_daemon/state/events.jsonl` | Event journal. Replayed on restart. **Flat, not per-config** — every deploy that doesn't set `journal_dir` shares this one file. Rows carry a `strategy_id` so the scheduler can tell them apart, but running two configs against one journal is still a bad idea: use `journal_dir = "..."` per deploy. Rotated segments sit beside it as `events-<UTC>.jsonl` and are part of the history — see [Journal rotation](#journal-rotation). |
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
  # ALL FOUR ARE MANDATORY when broker = "ibkr-live": the daemon
  # refuses to start with any of them unset or <= 0.
  # ALL FOUR ARE POSITIVE MAGNITUDES; <= 0 disables the gate.
  risk_max_position_per_symbol = 20_000, # USD notional, |pos| · mark
  risk_max_gross_exposure_usd  = 60_000, # Σ |price · pos|
  risk_max_daily_loss_usd      = 1500,   # POSITIVE; trips kill-switch
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

### What the live engine reproduces from your backtest

A strategy is gated by a backtest and then traded by the daemon. Where
the two size differently, the thing you deployed is not the thing you
validated — and historically that difference has been silent. This
table is the contract. Check any config against it with:

```
qe_daemon validate backtests/.../deploy_foo.qe
```

which loads the config, prints each leg's sizing mode and per-entry
budget, and exits without taking the instance lock, opening the
journal, or contacting the broker.

| `execution(...)` field | Live | Why |
|---|---|---|
| `capital` | ✅ reproduced | Split into per-leg budgets by the portfolio's `budget_weights` (or `weights` pre-v0.3.0). |
| `whole_shares` | ✅ reproduced | Floors each entry to an integer lot. |
| `entry_schedule` / `exit_schedule` | ✅ reproduced | Staged slices fire through the order scheduler. |
| `compound` | ⛔ **rejected at load** | The backtest scales orders by `equity/initial`; the live engine sizes from a static budget. Refusing beats trading a different book. Full support is a follow-up. |
| `commission_bps`, `slippage_bps`, `impact_bps_per_pct_adv` | ➖ ignored, correctly | Backtest cost models. Live pays the venue's real costs. |
| `fill_model` | ➖ ignored, correctly | Backtest fill convention. Live fills at the venue. |
| `record_forecast` | ➖ ignored | Diagnostic only. |

| `signal(... size)` form | Live | Why |
|---|---|---|
| scalar (`size = 2.5`) | ✅ reproduced | Fixed share count, capped by the leg budget. A `size = 1.0` leg against a large budget is warned about by name at startup — that combination usually predates v0.3.0 sizing. |
| `BudgetSized` (what `signalize_universe` emits) | ✅ reproduced | Spends the leg's whole budget at the entry mark. |
| per-bar expression (`size = 0.02 / rolling_vol(close, 20)`) | ✅ reproduced (v0.3.1+) | Evaluated at entry time against the same context as `entry` / `exit`. Before v0.3.1 this silently traded **one share**. |
| `InverseVolSized` (what `signalize_universe(... weighting = "inverse_vol")` emits) | ✅ reproduced | The allocator recomputes the weights each batch from `weight_expr`, so `validate` reports the capital and the allocator's bounds (`top_k`, `max_weight`, `target_gross`) rather than a static per-leg budget. |
| anything else | 🚫 cannot exist | The dispatch is an exhaustive `std::visit` over an overload set with no generic arm, so a new `SignalSize` alternative stops the build until every visitor handles it. It used to be a runtime `else`, which caught one site and let three others go stale. |

### Indicator warmup is capped at ~41 bars

The daemon fetches `"60 D", "1 day"` of history per symbol at startup
(`apps/daemon/main.cpp`), which is about **41 trading bars**. Any
indicator needing more never warms live and its leg sees NaN signals
— `rolling_vol(close, 20)` is fine, `sma(close, 50)` is not, and a
50/200 trend overlay cannot run at all. Design live strategies inside
that budget, or raise the fetch first.

### Picking risk limits

**A real-money venue will not start without all four (EPIC-88
T88.6).** `broker = "ibkr-live"` — or any broker name that does not
end in `-paper` — is refused at startup unless every one of
`risk_max_position_per_symbol`, `risk_max_gross_exposure_usd`,
`risk_max_daily_loss_usd` and `risk_max_orders_per_minute` is set to
a positive value, and the daily-loss one has a usable
`execution(capital = ...)` to anchor it. The refusal runs before the
daemon opens any socket to the venue, names every limit that is
missing, and exits 4.

The defect it closes (P3-10) is that all four default OFF and
`ibkr-live` was accepted with none of them armed. Nothing announced
that; the daemon connected, subscribed and would have discovered the
gap only by failing to reject the first order that needed rejecting.
An unset limit is not a wide limit — it is no limit.

Two escape hatches, both deliberate: a `-paper` broker gets a warning
listing the same gaps and starts anyway (a paper venue that refuses
half-written configs is a paper venue nobody uses), and
`qe_daemon validate <config.qe>` runs the identical gate without
touching the lock, the journal or the broker — so you can prove a
live config would be accepted before you point anything at IBKR.

**Sign convention (EPIC-83): all four knobs are POSITIVE
magnitudes, and `<= 0` disables the gate.** The three USD knobs are
validated at load time — a negative value is a hard config error, so
`qe_daemon start` refuses rather than starting with a silently
disabled gate. This is a change: `risk_max_daily_loss_usd` used to
be documented as negative while the `SafeBroker` backstop underneath
read it as positive, so one value armed one layer and disabled the
other. Write `1500`, not `-1500`.

`risk_max_position_per_symbol` is **USD notional**
(`|projected position| × latest mark`), not a share count.

A useful starting heuristic, **for a single-strategy / single-symbol
deploy**:

* **per-symbol position**: 1.2 × the largest USD notional the
  backtest ever held in one symbol. Catches a runaway loop without
  rejecting legitimate fills.
* **gross exposure**: 1.5 × `execution.capital`. The same multiple
  works for short books because the check uses `|pos · price|`.
* **daily loss**: roughly the 95th percentile of historical
  drawdowns in your backtest, as a positive dollar figure.
* **orders/minute**: 10 is comfortable for a daily/intraday strategy.
  Drop to 3 for once-a-day rebalances; raise to 60 if you're doing
  high-touch market making.

When in doubt: start tighter. The cost of a false reject is the
order doesn't go out (the strategy will re-fire next bar). The
cost of a false negative is a runaway loop on a real account.

> **Re-derive these after EPIC-83.** `signalize_universe` legs are
> now budget-sized at `1/(top_k + bottom_k)` of capital instead of
> one share each (see
> [`qe-language.md`](qe-language.md#leg-sizing-budget-sized-by-default-epic-83)).
> A book that used to run ~7 % deployed now runs ~100 % deployed, so
> a per-symbol cap that was dead headroom before is a live
> constraint now. Recompute both USD caps from the *new* intended
> notional, not from what the old deploy actually traded.
>
> The daily-loss gate deserves its own beat: on a book that was
> effectively 7 % invested it was unreachable in practice, and on
> most accounts it has never fired. At full deployment it is
> reachable. **The kill-switch is one-way** — a breach stops trading
> for the rest of the session and clearing it means
> `qe_daemon stop && qe_daemon start`. Pick a number you are willing
> to be stopped out at.
>
> **Know when it is evaluated: on every mark, since EPIC-88 T88.4.**
> The equity poller re-values the strategy book every 30 s and tests
> the threshold there, so a position bleeding through the limit trips
> within one poll interval **with no order in flight**. Before T88.4
> the only test was at submit time, which on a `rebalance = 20` deploy
> meant a breach could sit latent for ~20 sessions and then fire on
> the next order attempt, judged on the equity current *then*. The
> submit-time test in `PreTradeRisk::check()` (and the per-submit
> `SafeBroker` backstop) still run — they are now the backstop, not
> the whole mechanism.
>
> Three rules govern the continuous evaluation, all of them there
> because the kill is one-way:
>
> - **Two consecutive breaches trip, not one.** A single bad
>   valuation must not cost the trading day.
> - **Evaluation is skipped while the measurement is suspect** —
>   operator pause, watchdog soft-pause, or a failed broker poll.
>   Nothing can be submitted in those states anyway, so a trip would
>   only convert a reversible halt into an irreversible one. A skip
>   does *not* clear an accumulated breach.
> - **"Cannot value the book" is not "no loss".** A held symbol with
>   no mark makes the book unvaluable; that neither trips nor clears.
>   The daemon logs it at WARN on the edge, because a breaker that
>   cannot measure is not a breaker.
>
> `qe_daemon status` still recomputes `daily_loss_breached` on every
> poll with no side effects, and `scripts/verify-open-check.sh` alerts
> on it, so a breach the two-breach rule has not acted on yet is still
> visible. The journal gets a `daily_loss_breach` row on the first
> breach and another on the trip, carrying the P&L and the streak that
> the trip reason string (`daily-loss-kill`) cannot.

#### What the daily-loss number is measured against (EPIC-88)

`risk_max_daily_loss_usd` is a **strategy-scale** number. Both layers
that enforce it — `PreTradeRisk` (trips the kill-switch) and the
`SafeBroker` backstop (rejects the one order) — compare it against
*this strategy's own* marked-to-market P&L, anchored on
`execution(capital = ...)`.

Until EPIC-88 they compared it against the whole **account**: net
liquidation value on one side, `day_pl` on the other. On the paper
account that meant a $15 000 threshold measured against ~$1 002 648 of
account equity, on a book whose worst possible day was ~$8 333 gross —
a breaker that could not produce a true positive, and could produce a
false one from FX revaluation or another strategy on the same login.
`status` reported `daily_loss_armed: true` throughout.

Three consequences to know:

- **A daily-loss limit with no `execution(capital = ...)` refuses to
  start.** An armed threshold with nothing to measure is the defect
  this replaced; the daemon exits 4 and names the missing knob rather
  than starting inert.
- **`qe_daemon status` now spells the state out.** `daily_loss_state`
  is one of `absent` (no limit), `inert` (limit configured, nothing
  measuring it), `unanchored` (scoped, but the strategy book cannot be
  valued yet) and `armed`. `daily_pnl_usd` / `session_anchor_equity` /
  `strategy_capital_usd` are the strategy's; `account_equity_usd` /
  `account_daily_pnl_usd` are the account's and arm nothing.
- **The strategy book is built from this process's own fills.** It is
  not seeded from broker positions — adopting share counts without the
  cash that bought them would invent equity, and this number gates a
  one-way kill. Since EPIC-89 T89.11 a restart **restores** the book and
  its session anchor from the journal, as one unit, keyed to the session
  they were written in: cash, shares and anchor together, or none of
  them. So a daemon restarted mid-session keeps the day's loss instead
  of re-anchoring flat.

  Two residual cases. A checkpoint written under a different
  `execution(capital = ...)` is refused, and the daemon measures from a
  flat book until the next 16:00 ET roll — the pre-T89.11 behaviour,
  now only on a config change. And a daemon that dies between a fill
  and its journal append restores the pre-fill ledger. Both are stated
  at WARN on startup. Restarting outside market hours is still the
  calmer option, but it is no longer the difference between a working
  daily-loss guard and an inert one.

  (This bullet used to carry a second warning, that an evening restart
  also dropped the orders the close eval had queued. It no longer does
  — see [staged entry / exit](#) in the feature list below.)

## Upgrading an existing deployment

Read this **before** you restart a running daemon onto a newer build.
Three 0.3.4 changes can turn a config that started yesterday into one
that refuses today. All three are refusals, not silent behaviour
changes, and all three exit before a socket to the venue is opened.

The whole check is one command, and it contacts no broker:

```bash
qe_daemon validate path/to/live_config.qe   # exit 0 = deployable
```

Run it against the *currently deployed* config, on the *new* binary,
while the old daemon is still up. If it exits 0, restart normally.

### 1. `rolling_vol` on a bare price is now fatal (BREAKING)

**A daemon restarted on a 0.3.4-or-later build refuses to start on a
config that applies `rolling_vol` to a bare `close` / `open` / `high` /
`low`.** `qe_daemon validate` and `qe_daemon start` both exit **4**.

`rolling_vol(close, 20)` is the standard deviation of the dollar price
level, not of returns. On a mixed-price universe
`rolling_vol(close, n) * -1.0` ranks the cheapest names rather than the
calmest, and it ranks a name moving 4 % a day above one whose returns
are constant. This warned at load time for a full release and a paper
deployment ran on it for six weeks, which is why the warning became a
refusal. Full measurements are in
[Safety & known limitations](safety-and-limitations.md#semantic-traps).

`qe_run` is deliberately unaffected — reproducing what a legacy config
actually did has to keep working, or the record becomes unauditable.

**The remedy, in the order you should consider it:**

```qe
// Before — refused at the live boundary
rolling_vol(close, 20)

// Option A — you meant return volatility. A DIFFERENT factor.
realized_vol(close, 20)

// Option B — you meant dollar dispersion. Exact identity, same numbers.
rolling_vol(price_level(close), 20)
```

- **Option B is a no-op and is the one that unblocks a restart.**
  `price_level(x)` returns `x` and allocates no state, so the
  arithmetic is byte-for-byte unchanged: same values, same rankings,
  same orders. **You do not need to re-run your gates.** The config
  simply now says on the page that dollar dispersion was the intent.
- **Option A is a different strategy.** `realized_vol` measures
  something else and will select different names. Treat it as a new
  deployment with new gates, never as a rename.
- **Option C — `--allow-price-level-vol`.** Starts anyway and prints
  the entire refusal at ERROR on every startup. It buys you a running
  daemon during an incident; it is not a resolution, and the log will
  not stop saying so.

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
is exactly the pre-0.3.4 behaviour. Read from the replay code, not
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
| Stop the daemon submitting anything new, from CLI | `qe_daemon kill --yes` (EPIC-88 T88.11). **One-way** — nothing clears it but a restart. Blocks new submissions and drops queued staged slices; working orders are untouched. |
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
   blocked and queued staged slices are dropped. It cancels
   nothing at the venue — that is decision D1 and it is
   deliberate. Without `--yes` you get a confirmation prompt; in a
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

| Wall-clock offline | Watchdog state | F6 `broker:` badge | Effect |
|---|---|---|---|
| ≤ 3 s | Healthy | `connected` | Normal — TCP heartbeat blip absorbed |
| 3 – 30 s | **Paused** | `paused · Ns` (amber) | New orders reject (`PreTradeRisk` gate). Open orders untouched. Auto-resumes on reconnect. |
| > 30 s | **Tripped** | `TRIPPED (ibkr_disconnect)` (red) | KillSwitch tripped; pending staged slices dropped locally. **Open orders at the venue are NOT cancelled** — a cancel needs the broker link, which is the thing that just died. The journal gets the count of orders left working, or `unknown` when even that could not be read. Cancel them by hand in TWS. Operator must restart. |

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
  ("ibkr_disconnect")`, plus a local drop of every unfired staged
  slice. It does **not** cancel at the venue: it never could (the
  link whose loss triggered it is the link a cancel would need),
  and under EPIC-88 D1 it must not. EPIC-88 T88.2 deleted that
  loop; what it writes instead is the honest count of orders left
  working, or "unknown" when it cannot ask. The hard trip is
  sticky for the process lifetime; restart the daemon to clear.
- **Broker reconciliation on restart** —
  `qe::live::reconcile_positions()` compares the journal-replayed
  `in_position[symbol]` map against
  `broker_session->list_positions()` and reports per-symbol drift.
  Since EPIC-89 T89.10 that report **blocks new order submissions**
  until an operator acknowledges it, and so do a `position_drift`
  latched by the running reconciler and an uncorroborated fill. Run
  `qe_daemon reconcile` for the report and `qe_daemon reconcile-ack
  <block-id> --decision=<d>` to resume; the block is journaled, so a
  restart does not clear it. The ack records your decision and adopts
  nothing. Cancels and `cancel-all` are never gated — but **exits are**,
  so a block leaves any held position unmanaged until you clear it.
  That is the trade: an exit is sized against the position whose size
  is what the block is about.
- **F6 TRADE daemon-mode panels** (EPIC-67) — Working Orders /
  Executions / Account / Order Log read from the daemon's control
  socket (`orders` / `positions` / `equity` / `log_tail`) when a
  daemon is attached. `DaemonOrderCache` polls every 3 s and
  reshapes the JSON into the same `OrderSnapshot` shape the
  local-broker panels consume. The per-order Cancel button is
  disabled in daemon mode (there is still no `cancel_order` verb);
  to clear the whole book use `qe_daemon cancel-all`, the F6
  **CANCEL ALL** control (T88.1), or the broker's own UI. Trip kill-switch is **not** a substitute: it
  cancels nothing at the venue.
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
  is journaled and replayed on restart, so a fired slice is never
  double-submitted. See
  [Staged entry / exit](staged-entry.md) for the full reference
  including the F6 TRADE STAGED panel.

  > **A graceful restart keeps the queue. Only a kill switch clears it.**
  > `qe_daemon stop` leaves Pending slices alone; they rehydrate on the
  > next start and fire in their own windows. A slice that comes back
  > **after** its window — a weekend outage, a long upgrade — is retired
  > by `tick`'s window-miss enforcement and journalled `window_miss`,
  > never submitted late at whatever price is on screen when the daemon
  > wakes up.
  >
  > It did not always work this way, and the history is worth one
  > paragraph because the old behaviour is the kind that looks prudent.
  > T83.8 cancelled every Pending slice on shutdown, against the hazard
  > of a rehydrated slice being market-ordered hours out of window. That
  > hazard was real — and was already closed by the window-miss
  > enforcement in `32cb945`, which is an ancestor of the `4b69cde` that
  > added the cancel. The insurance was written against something the
  > sibling commit had fixed.
  >
  > What it cost was not hypothetical. A daily strategy queues its
  > slices at the 16:00 close for the next open, so **every restart in
  > the window this runbook calls safe** — outside market hours — threw
  > the session away silently. Observed 2026-07-31: an evening restart
  > to pick up a one-line fix dropped ten slices, and there was no
  > recovery, because `backfill` correctly refuses a close already in
  > the eval index. Reversed 2026-07-31 by operator decision.
  >
  > A kill-switch trip still cancels everything. That is what it is for.

- **Eval self-healing** (EPIC-75) — every close eval journals a
  canonical `eval_round`; same-close re-evals are idempotent
  (never double-order); a background sweep alerts on missed
  closes (`eval_missed`); and a missed close can be **backfilled
  same-evening** — automatically on broker reconnect (TWS
  `1101`/`1102`) or manually via `qe_daemon backfill --latest`.
  F6 TRADE surfaces the latest round + alert count on the EVAL
  line. See [Eval self-healing](eval-self-heal.md).

**Not yet wrapped:**

- **`unkill`** — implemented on the control socket only as a
  *refusal*, and that is the whole feature: `KillSwitch` is
  one-way by design, so the verb exists to return a precise "restart
  the daemon" message rather than an "unknown verb" one. `qe_daemon
  unkill` prints the same thing and exits 2. There is nothing here
  to wrap.

  `kill` / `pause` / `resume` were in this list until EPIC-88
  T88.11 and `cancel-all` until T88.1; all four are now real
  subcommands, and the dashboard sends `kill` from F6 SAFETY
  (T88.10).
- **Per-order cancel from the dashboard in daemon mode** — the
  daemon exposes `cancel_all` (T88.1, reachable from the CLI *and*
  the F6 CANCEL ALL control) but still no `cancel_order`, so the F6
  Working Orders panel's per-order Cancel button remains disabled
  in daemon mode. All-or-nothing is the only granularity there is. The tooltip
  (the `"This order belongs to the daemon…"` `SetTooltip` in
  `apps/dashboard/gui_screen_trade.cpp`) points at the broker's own
  UI. Note the standing rule it states is still true: **no
  kill-switch cancels working orders**, in either process — only
  the explicit `cancel-all` verb and TWS do.
- **Alpaca daemon support** — only `ibkr-paper` / `ibkr-live`
  start the runtime. `broker = "alpaca-..."` parses but exits 4
  with a clear error.

See [`qe-daemon-smoke.md`](qe-daemon-smoke.md) for the manual
IBKR-Gateway smoke checklist you should run after first install
and again before any live promotion. The smoke was verified
end-to-end against IB Gateway 151 paper on 2026-06-04.

[live-trading-safety.md]: live-trading-safety.md
