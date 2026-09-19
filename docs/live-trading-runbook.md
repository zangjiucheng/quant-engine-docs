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

  # The venue's own nightly outage, declared so it stops costing a
  # restart. Both or neither; off by default.
  maintenance_window_utc_minute = 1425,  # 23:45 UTC
  maintenance_window_minutes    = 45,
)
```

**The maintenance window is the knob that stops IB Gateway's nightly
exit from killing the daemon.** Gateway ends its session once a day;
the watchdog escalates that to a one-way kill in 27 seconds, and a
kill needs a restart plus a reconcile ack to clear — so without this
a scheduled vendor event costs a manual intervention every night.

Inside the window a disconnect still soft-pauses, probes, journals
and counts misses. It just does not **trip**. Still down when the
window closes and it trips on the next poll: deferred, not cancelled.
Set it knowing it weakens the control that catches a real venue
outage, on a schedule, every day. See
[the `live(...)` reference](qe-language.md) for the ranges.

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

### Indicator warmup is sized from your config

The fetch is **derived, not fixed**. `apps/daemon/warmup_plan.hpp:120`
takes `min_history` across every leg's entry, exit and sizing
expression — descending into cross-sectional sites — and
`:164` converts that bar count into an IBKR `durationStr` with a
calendar buffer. `sma(close, 50)` warms fine, and so does a 50/200
trend overlay.

There is no budget to design inside, but there is a failure to know
about: **a short delivery refuses the start.** If the venue returns
fewer bars than the plan requires, the daemon logs the refusal and
exits **4** (`apps/daemon/main.cpp:1596-1602`). It does not start
degraded, and it does not run a leg on NaN. On a clean start you get

```
daemon: historical warmup = N bars across M symbols (min K per symbol, needed R)
```

Run **`qe_daemon validate`** to see the requirement without a broker
— it prints what the config demands before you depend on the venue
supplying it.

> This section previously said warmup was capped at ~41 bars from a
> hardcoded `"60 D", "1 day"` fetch, and told authors to design
> around it. That cap is gone. If you cut a strategy down to fit it,
> the constraint no longer applies.

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
> reachable — *provided the ledger it measures is tracking the book*,
> which on the live deployment it was not, so the breaker could not
> fire at all. That is PER-99 and it has its own subsection below;
> check `daily_loss_state` before you trust any of this paragraph.
> **The kill-switch is one-way** — a breach stops trading
> for the rest of the session and clearing it means
> `qe_daemon stop && qe_daemon start`. Pick a number you are willing
> to be stopped out at.
>
> **Know when it is evaluated: on every mark, since EPIC-88 T88.4.**
> The equity poller re-values the strategy book every 30 s and tests
> the threshold there, so a position bleeding through the limit trips
> within one poll interval **with no order in flight** — a book the
> ledger holds, that is. What the poller re-values is the ledger, not
> the account, so shares the ledger never booked bleed invisibly at
> any interval. Before T88.4
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
  measuring it), `untracked` (the ledger being measured is not the
  strategy's book — see below), `blind` (a held symbol has no mark, so
  the book cannot be valued at all), `unanchored` (scoped, but no
  session anchor yet) and `armed`. Only `armed` means the breaker can
  fire; `daily_loss_armed` is now false for every other one.
  `daily_pnl_usd` / `session_anchor_equity` / `strategy_capital_usd`
  are the strategy's; `account_equity_base` / `account_daily_pnl_base`
  are the account's and arm nothing.

  **Those two are in the account's BASE currency, not USD**, and
  `account_currency` beside them says which (`CAD` on the live paper
  account). They were named `_usd` until 2026-08-11 and nothing in the
  tree converts anything, so the name was the only clue and it was
  wrong. Note that `account_gross_usd` keeps its suffix and is *not* the
  same unit: it is summed from position marks, and the positions are US
  equities. Two of the three account figures are CAD and one is USD —
  which is why they must not be compared or subtracted without an FX
  rate the daemon does not have.
- **`untracked` means the ledger is empty and should not be.** An empty
  strategy book values cleanly to its opening capital, so the day's
  loss reads `$0` forever and the kill cannot fire — which is what the
  live daemon did for its whole deployment while holding ~$7,430 of
  stock. It is raised either by a sell the ledger refused because it
  held nothing to back it (`strategy_unbacked_sells`) or by a startup
  comparison against the strategy's own holdings
  (`strategy_untracked_holdings`, logged at ERROR). That comparison
  reads two sources: the broker's book with your declared external
  baseline subtracted, and this daemon's own journalled `position` rows
  — the second because a baseline stale enough to claim the strategy's
  own shares would empty the first. `verify-open-check.sh` raises it as
  a problem, so it reaches the morning notification. The ledger is
  fill-driven and cannot repair itself — it does not know the cost
  basis of shares it never saw — so this state persists until the book
  is rebuilt. **`qe_daemon book rebuild` is that repair**; see below.
- **The strategy book is built from this process's own fills.** It is
  not seeded from broker positions — adopting share counts without the
  cash that bought them would invent equity, and this number gates a
  one-way kill. Since EPIC-89 T89.11 a restart **restores** the book and
  its session anchor from the journal, as one unit, keyed to the session
  they were written in: cash, shares and anchor together, or none of
  them. So a daemon restarted mid-session keeps the day's loss instead
  of re-anchoring flat.

  Three residual cases. A checkpoint written under a different
  `execution(capital = ...)` is refused, and the daemon measures from a
  flat book until the next 16:00 ET roll — the pre-T89.11 behaviour,
  now only on a config change. A daemon that dies between a fill
  and its journal append restores the pre-fill ledger. Both are stated
  at WARN on startup. Restarting outside market hours is still the
  calmer option, but it is no longer the difference between a working
  daily-loss guard and an inert one.

  The third is the one that actually happened, and no restart fixes it:
  a ledger that never held the fills in the first place restores
  faithfully as empty and stays empty. The restore cannot tell "empty
  because the strategy is flat" from "empty because these fills predate
  the checkpointing that would have persisted them" — which is what the
  live daemon was, for its whole deployment, while holding ~$7,430 of
  stock. That is the `untracked` state above, and `qe_daemon book
  rebuild` is the only thing that clears it.

  (This bullet used to carry a second warning, that an evening restart
  also dropped the orders the close eval had queued. It no longer does
  — see [staged entry / exit](#) in the feature list below.)

#### Rebuilding the ledger — `qe_daemon book`

When `daily_loss_state` is `untracked`, the breaker is measuring an
empty ledger and the kill cannot fire. The journal is what repairs it:
every fill the daemon booked was written there by the same callback
that fed the ledger, from the same quantity and price, so replaying
those rows back through the ledger reproduces what it should have held.

```bash
qe_daemon book                                # what the ledger holds now
qe_daemon book rebuild                        # DRY RUN: reconstruction + a token
qe_daemon book rebuild --token=<t> --yes      # commit it
```

The dry run writes nothing — not the ledger, not the anchor, not the
journal — and is served on the read-only socket. The commit is not.

Six things to know before you run the commit:

- **Today's P&L restarts from zero.** The session anchor is adopted
  *with* the book and becomes the rebuilt book's value at that moment.
  It has to: leaving the anchor at `execution(capital = ...)` would
  report the strategy's whole since-inception return as today's loss,
  which on a strategy that is down is a false trip on a one-way kill.
  Whatever this strategy made or lost earlier today is **not**
  recoverable — the ledger never saw it.
- **Run it with nothing in flight**, before the open or after
  `qe_daemon cancel-all`. A fill landing mid-rebuild would be booked
  into the ledger being replaced and lost, so the verb refuses while
  any order is working or any staged slice is pending — and refuses
  too if the order link cannot say.
- **It refuses a ledger that already holds something.** This is a
  repair, not a re-baseline: replaying over a healthy ledger would
  discard whatever fills the journal no longer has.
- **It refuses a ledger that has not *lost* the book** — that is, one
  where `daily_loss_state` is not `untracked`. Empty is not lost: a
  strategy out of the market has a flat ledger that is entirely
  correct, and on one of those the replay changes no cash and no
  shares. The only thing left that the commit *would* do is move the
  session anchor, which forgives today's loss and hands the one-way
  kill a fresh full allowance — repeatably. The same refusal covers a
  replay that reproduces the current ledger exactly: nothing to repair,
  so nothing to commit.
- **It refuses a reconstruction worth $0 or less** at current marks.
  The anchor *is* that value, and a non-positive anchor is ignored by
  the tracker (as it is by the restart path), so committing would adopt
  the shares and leave the anchor where it was. A reconstruction worth
  nothing means the journalled fills cost more than
  `execution(capital = ...)`: legs sized off the whole account, or
  another deployment sharing this `journal_dir`.
- **It re-runs the detector afterwards rather than declaring victory.**
  If the journal was missing the fills too, `untracked` stays lit and
  the reply says how many symbols are still unaccounted for. Treat that
  as "the breaker is still not measuring", not as a partial success.
  And if `list_positions()` does not answer, the comparison is **not**
  re-run at all — with no broker side it would compare the journal
  against a book rebuilt from that same journal and come back clean by
  construction. The previous count stands, the reply says so, and a
  restart once the link is back is what re-measures it.

The token covers both the reconstruction and the ledger it would
replace, and the commit recomputes it — so a token that was printed
before a fill landed will not commit, and you get to read the new
reconstruction first.

Order matters against the external-baseline cut-over in the next
section: **rebuild first, then adopt a baseline.** The baseline verbs
refuse against an empty ledger, because an empty ledger would attribute
the whole account as external and let the strategy buy a second book on
top of the one it already holds.

#### Whose shares the caps count — `qe_daemon baseline`

`risk_max_gross_exposure_usd` and `risk_max_position_per_symbol` are
sized from the strategy's own capital. Until you say otherwise they are
checked against the **whole** IBKR account, so anything you hold in that
account by hand counts against the strategy's caps.

Know the shape of the failure, because it does not look like one. Exits
are unaffected — an exit reduces risk, the risk-reducing test is
deliberately still account-scoped, and no cap ever refuses a sell for
being over-scope. Buys are what get refused. The strategy therefore goes
quietly **sell-only** while reporting healthy: no error, no kill-switch,
just entries that stop appearing. On the live account on 2026-08-07 that
was $10,986 of account measured against a $10,000 cap, $3,556 of it
bought by hand.

**A baseline is one number per symbol: how many of the account's shares
are yours.** The daemon subtracts it from the account before the caps
measure, before the engine decides what it is allowed to sell, and
before either reconciler calls a difference drift. It is journalled,
survives restarts, and carries a revision number. With none declared —
the state every daemon starts in — nothing is subtracted and the
behaviour is exactly what it was before, which is why `status` calls
that state `⚠ ACCOUNT` rather than something reassuring.

##### The order, and it is not negotiable

```bash
qe_daemon status                  # EXPOSURE block: which book the caps measure
qe_daemon book                    # is the ledger tracking? (previous section)
qe_daemon book rebuild            # ...and repair it FIRST if it is not
qe_daemon baseline                # what is declared right now
qe_daemon baseline propose        # DRY RUN: the split, the totals, a token
#   >>> read the table against TWS here <<<
qe_daemon baseline adopt --token=<t> --note="hand-bought FTNT/INTC" --yes
qe_daemon stop && qe_daemon start # so the ENGINE moves with the caps
```

**Do the whole thing before the open**, or after `qe_daemon cancel-all`.
`propose` and `adopt` both refuse while anything is working, because a
share that fills between the proposal and the commit lands in the
account before it lands in the ledger — and that difference is exactly
what the verb would attribute to you, permanently.

**The restart at the end is part of the procedure, not tidiness.** The
caps pick the new split up immediately; the engine does not. Its legs
were seeded at start-up through whatever split was in force then, so
until it restarts it still believes it may sell shares the baseline has
just declared yours (PER-61). The report says so after every adopt.
Restarting before the next rebalance is what closes it.

##### Reading the table

`propose` prints one row per symbol and derives `ext = broker − strat`,
floored at zero, where `strat` is the strategy's own fill-driven ledger.
Two of the eleven rows the live account produced, and its totals:

```
    sym       broker   strat     ext    strategy $    external $     running $
    FTNT          21       5      16       $798.20     $2,554.24     $4,019.75
    INTC           8       0       8         $0.00       $813.20     $4,019.75
    TOTAL                                $7,429.95     $3,556.11

  gross      $7,429.95 of a $10,000.00 cap — $2,570.05 of headroom
             the account measures $10,986.06 against the same cap today
```

**Check the `strat` column against TWS. Nothing checks it for you.**
That is the whole reason `propose` is a dry run. The refusals below
catch a ledger that is *empty* or has *holes*; none of them catches a
ledger holding the wrong *number* of a symbol it does carry, and the
staleness auditor cannot see it either — a baseline derived from the
ledger agrees with the ledger by construction. Every share the ledger is
short becomes a share attributed to you, whose notional the strategy
gets back as headroom it can spend. A row annotated `⚠ ledger holds more
than the account` or `⛔ the ledger claims the OPPOSITE side` is telling
you the two disagree before you commit anything; read those rows first.

Two smaller things in that output. `no mark` on a row means all three
totals are lower bounds — the gate skips unpriced rows in its own gross
walk and these numbers do the same. And headroom on a thin name can be
a couple of dollars: the per-symbol cap is checked against the
*post*-fill line, which on the measured account left a $1,700 cap
admitting a buy at $1,698.03.

##### What each refusal means

`propose` (and `adopt`, and `set`) refuse rather than derive from
something they cannot trust. Exit code 3, and the message is the
diagnosis:

| The message says | What it means | What to do |
|---|---|---|
| `no strategy ledger — execution(capital = ...) did not scope one` | There is no strategy book to subtract, so `broker − ledger` would hand you the whole account. | Fix the config, restart. |
| `could not list account positions` | The position link did not answer. Silence is not an empty account. | Retry when the link is back. |
| `the strategy ledger is EMPTY while the account holds N symbol(s)` | **The PER-99 state**, and the one that matters. Deriving here attributes everything to you, the caps then measure $0, and the next rebalance buys a second full book on a cap that cannot fire. | `qe_daemon book rebuild` first — previous section. |
| `the journal's strategy_book checkpoint was NOT adopted by this process` | The checkpoint was written under a different `execution(capital = ...)`, or is malformed, so the ledger holds only what *this* process filled. Editing the capital alone gets you here. | Restore the capital the checkpoint was written under and restart, or `book rebuild` under the new one. |
| `journalled position rows say the strategy is in N symbol(s) its ledger is flat on` | The ledger has holes: this daemon's own journal recorded fills for a symbol the ledger shows nothing in. A symbol *you* hold and the strategy has never traded is **not** this — that one never wrote a position row, and declaring it is what the verb is for. | `qe_daemon book rebuild`. |
| `the ledger has refused N sell(s) it could not back` | The ledger has already said out loud that it is behind the engine, so its share counts are a lower bound. | `qe_daemon book rebuild`. |
| `could not list working orders` | Unknown whether anything is in flight. A leg that printed but whose execution report has not landed appears in the account and not in the ledger. | Retry when the order link is back. |
| `N order(s) working ... M staged slice(s) pending` | Something is in flight. | Before the open, or after `qe_daemon cancel-all --yes`. |
| `the risk gate's projected account book disagrees with the venue on X` | Nothing is working, so the two snapshots this baseline would be derived from describe different moments. | Re-run in a minute. If it persists, `qe_daemon reconcile` owns that disagreement. |

Two more on `adopt` alone. It **recomputes** the split and refuses a
token that no longer matches, so a proposal that went stale while you
were in TWS is refused rather than applied — re-run `propose` and read
the new table. And it refuses if its journal row did not reach the disk,
because a claim that is not on disk is a claim that disappears at the
next restart.

`baseline set FTNT=16 INTC=8 --yes` takes the **same** refusals. It
derives nothing, so the arithmetic argument does not apply to it — but
the hazard does, and a ledger that has lost the book is exactly the
state in which you cannot know what to type. Its counts are absolute and
replace the whole baseline: a symbol you leave out is a symbol you own
none of.

##### What the cut-over leaves behind

The daemon's journalled `position` rows record what its position tracker
held when each fill landed. Rows written **before** the cut-over are
account-scoped; the tracker is strategy-scoped after it. Nothing
rewrites the old ones — the journal is append-only, and rescoping
history is a migration this release deliberately does not attempt.

What you will see, on the restart at the end of the procedure and on
every restart after it: the startup reconcile compares each journalled
row against `broker − your baseline`, so on a symbol whose last row
predates the cut-over the two disagree **by exactly the quantity you
declared** and that raises a reconcile block. With the numbers above,
FTNT's row says 21 against an attributed 5.

That is not drift and there is nothing to repair. `qe_daemon reconcile`
prints the venue's own quantity beside the strategy's, so you can see
the block is the declared split rather than a real difference; confirm
that, then `qe_daemon reconcile-ack <id> --decision=broker_is_correct`.

**The window is per symbol and closes at that symbol's next fill**, which
rewrites the row in the new scope — not at the next restart. A symbol
you declared and the strategy then never trades keeps its stale row
indefinitely, and because an ack clears the block rather than the
condition, it blocks again at every start until it trades. Know that
before you decide the cut-over did not take. After the ack, `qe_daemon
status` keeps the RECONCILE section at `⚠` and lists what you
adjudicated on its `acked` line — that is the condition still standing,
not a second finding.

One narrower case, for completeness: if `list_positions()` fails at
startup the daemon seeds from those same journal rows and rebuilds the
account book as `strategy + baseline`, which on a stale row
double-counts. It errs restrictive — the caps over-state the strategy
— and `PreTradeRisk`'s 60-second reseed corrects it once the link is
back.

##### Reverting

```bash
qe_daemon baseline disable --note="why" --yes
```

Immediate, in-process, no restart, and it takes **no** preconditions —
account scope is the state the gate has always been able to run in, so
the escape hatch works when everything else is refusing. The caps go
back to measuring the whole account, which is the PER-71 state above,
so expect the sell-only behaviour back with it. The cleared claim is
journalled as cleared and is not resurrected by a restart;
re-establishing goes back through `propose` / `adopt`.

The engine, again, does not move until it restarts. After a `disable`
that lag is in the safe direction — the engine goes on treating only
the strategy's shares as sellable — so there is no hurry.

##### When `baseline_stale` fires

Each 60-second reconcile pass compares `account − your baseline` against
the strategy's own ledger. Two consecutive disagreements on a symbol
raise a **`baseline_stale`** finding, which is a reconcile block: like
drift, it stops all new submissions — entries *and* exits — until you
acknowledge it. `qe_daemon status`'s RECONCILE block shows how many
symbols are stale **now** alongside the rising-edge total since start —
the two are separate numbers, so the section returns to `clean` when the
audit is running and the books agree again. When the audit is *not*
running it says `BASELINE AUDIT NOT RUNNING` instead, and that is not the
same claim: see the two silent states at the end of this section.
`qe_daemon reconcile` prints the finding.

**It cannot tell you which side is wrong, and it does not pretend to.**
An undeclared hand trade and a fill the strategy's ledger never booked
produce an identical signature — same symbol, same delta, same
direction — and no number this daemon holds separates them. The finding
prints all four quantities (venue, claimed, implied strategy share,
ledger) and then sends you to TWS, because the venue is the only
evidence that decides it.

Once you know which side is wrong:

- **you traded it by hand** → restate the claim with
  `qe_daemon baseline set ... --yes`, listing **every** symbol you hold,
  not only the one that fired: `set` replaces the whole baseline rather
  than merging into it, so a symbol left out silently becomes one you own
  none of. `qe_daemon baseline` first, to read what is declared now. Or
  skip the typing and re-derive the whole split with `propose` /
  `adopt`, once the ledger is right;
- **the ledger is behind** → `qe_daemon book` to read it, then
  `qe_daemon book rebuild`;
- either way, `qe_daemon reconcile-ack <id> --decision=resolved_externally`
  once the two agree again — or `--decision=accept_risk` to resume
  without deciding. An ack releases the block and changes neither book.

It stays silent in two states rather than guessing, and neither is a
clean bill of health: when no baseline is declared (there is no claim to
audit, and the comparison would degenerate into PER-99's signal, which
has its own detector) and when the ledger has declared it lost track (a
book that says it is not tracking cannot testify about anything else).

Both states drop the stale-symbol count to zero, so whenever a finding
still stands, `status` says `BASELINE AUDIT NOT RUNNING` rather than
`audit clear now` — a zero it did not measure is not a zero it found.
Reaching that state from a live finding is easy and is on the documented
path: ack the block to resume, then `baseline disable` (or let one
unbacked sell flip the ledger to lost-track), and the disagreement is
still there with nothing watching it. `qe_daemon book rebuild` if the
ledger is what stopped; `propose` + `adopt` to re-declare the split.

"Still stands" is answered by the reconcile gate, not only by this
process's alert counter, and the difference is a whole class of
incident. No block is carried across a restart: the new process
re-measures, comparing a fresh `list_positions()` against the replayed
journal, and raises a new block only if the disagreement is still
there. So an unacknowledged block returns when the condition does — but
if that startup query comes back inconclusive, the reconcile is skipped
and the block does not return at all, having been neither cleared nor
confirmed. Ack it at 09:05 instead and the ack clears the *block*, not
the condition. In every one of those paths no reconcile pass in the new
process ever raised an edge, so the counter reads 0 — and if the ledger
also came up lost-track, every field that existed before this check
reads exactly like a healthy daemon. The daemon publishes the join as
`baseline_audit_blind` on `status`, the RECONCILE section refuses to
read `clean` while it holds, and the dashboard's F6 SAFETY row lists it
under `UNMEASURED`.

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

### Changing a config without restarting (PER-228)

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

**A daemon restarted on a 0.4.0-or-later build refuses to start on a
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
| Declare which of the account's shares are yours (once, and again after you trade by hand) | `qe_daemon baseline propose` → check the table against TWS → `qe_daemon baseline adopt --token=<t> --yes` → restart. Read [Whose shares the caps count](#whose-shares-the-caps-count-qe_daemon-baseline) first; run it before the open. |
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
count](#whose-shares-the-caps-count-qe_daemon-baseline) is the fix.
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
`log_tail`, `reconcile_status`, `book` and `baseline`
(`apps/daemon/read_only_control_handler.hpp:87-97`). Everything else is refused
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
daemon" for a reason nowhere on screen; since PER-65 it is visible —
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

`python/qe_daemon/` is a standard-library-only client for the control
socket. It links nothing and builds nothing, so it runs on a machine
with no engine on it — the same property that lets the dashboard attach
to a remote daemon.

```python
import sys; sys.path.insert(0, "python")
from qe_daemon import DaemonClient, DaemonRefused

c = DaemonClient("/run/user/1002/qe_daemon.sock")
print(c.status()["slices_pending"])
print(c.reload_check(config="live_config.qe")["changes"])
```

Read verbs hang off the client; mutating verbs hang off
`c.control` — `c.control.pause()`, `c.control.reload_apply(...)`. That
mirrors the daemon's own `book` / `book_rebuild` split so a mutation is
visible at the call site.

**The split is naming, not enforcement.** The daemon allow-lists the
read verbs by name and refuses everything else per connection
(`apps/daemon/read_only_control_handler.hpp`); `c.control.kill()`
against a read-only socket raises because the daemon said no, not
because the client declined to send it. That is the property worth
having — a client cannot exceed the policy by being written badly — and
it only holds because the policy is not in the client.

Errors are typed: `DaemonRefused` (the daemon answered no, carrying its
own message), `DaemonUnreachable` (socket), `DaemonProtocolError` (the
reply did not match the envelope). Nothing returns a default. A verb
your daemon predates raises `DaemonRefused` naming it, which is how you
feature-detect.

To re-pin the client against a running daemon after changing either
side — read verbs only, safe against a live book:

```bash
ssh <host> 'cd ~/quant-strategy && python3 - --socket /run/user/1002/qe_daemon.sock' \
    < scripts/per240-pin-wire-format.py
```

## What ships, what doesn't

The end-to-end wire-up landed (event loop → router → risk →
journal → socket → IBKR), and the `epic/62-finish` follow-ups
closed the remaining EPIC-62 acceptance criteria except for
two operator-side gaps:

**Shipped:**

- **IBKR `reqHistoricalData` warmup** — the daemon calls
  `IbkrConnection::request_historical_bars` for **every** subscribed
  symbol after the handshake (EPIC-66 follow-up), with the duration
  **derived from the config's `min_history`** rather than a fixed
  `"60 D"` (`apps/daemon/warmup_plan.hpp:120,164`). Pre-fix only the
  first symbol was warmed, which made cross-sectional deployments
  start with NaN factor values for 29 of 30 symbols. Logs as
  `historical warmup = N bars across M symbols (min K per symbol,
  needed R)`.

  **A per-symbol failure is fatal.** A fetch that fails is recorded
  as 0 bars for that symbol, which is a short delivery, and the
  daemon refuses to start — exit 4
  (`apps/daemon/main.cpp:1596-1602`). This bullet said the opposite
  until 2026-08-11; an operator restarting during an IBKR pacing
  violation should expect no daemon rather than a degraded one.
- **3-state disconnect watchdog** — `qe::live::DisconnectWatchdog`
  (EPIC-70) polls `IbkrConnection::is_connected()` every 1 s. After
  ≥ 3 consecutive misses it enters **Paused**: fires
  `on_soft_pause("ibkr_disconnect_soft")` → `PreTradeRisk::
  set_watchdog_pause` → new orders reject; open orders untouched.
  Reconnect within the 30-tick trip window auto-resumes
  (`on_resume` → `clear_watchdog_pause`). At 30 consecutive misses
  the watchdog escalates to **Tripped** — `KillSwitch.trip
  ("ibkr_disconnect")`. Unfired staged slices are **preserved**,
  and the daemon logs `watchdog: N staged slice(s) PRESERVED across
  the kill`; a restart re-evaluates them, firing those still inside
  their window and retiring the rest as `window_miss`. It does
  **not** cancel at the venue: it never could (the
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

  > **The queue survives both a graceful restart and a kill.**
  > `qe_daemon stop` leaves Pending slices alone; they rehydrate on the
  > next start and fire in their own windows. So does a kill-switch
  > trip — the killed branch dispatches nothing and destroys nothing
  > (`src/live/order_scheduler.cpp:521`). Earlier revisions of this
  > page said a kill cleared the queue; it did once, and that is
  > exactly the behaviour the paragraph below records as removed. A slice that comes back
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
