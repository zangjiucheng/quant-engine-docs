# Daemon configuration and accounting

Configure the strategy, risk limits, ledger and account baseline here.
For installation, deployment, daily commands and troubleshooting, use the
[daemon runbook](live-trading-runbook.md).

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
> fire at all. That case has its own subsection below;
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
  — see [staged entry / exit](live-trading-runbook.md#) in the feature list below.)

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
just declared yours. The report says so after every adopt.
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
| `the strategy ledger is EMPTY while the account holds N symbol(s)` | **The dangerous state**, and the one that matters. Deriving here attributes everything to you, the caps then measure $0, and the next rebalance buys a second full book on a cap that cannot fire. | `qe_daemon book rebuild` first — previous section. |
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
back to measuring the whole account, which is the state above,
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
audit, and the comparison would degenerate into the empty-ledger signal, which
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
