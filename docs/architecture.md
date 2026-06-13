# Architecture

This page is the "how does it all fit together" entry point —
the binaries that ship, how they talk to each other, and the
shape of the data that flows through them.

## What ships

```mermaid
graph TB
    subgraph "User-launched binaries"
        DASH["QE Dashboard.app<br/>(GLFW + ImGui + ImPlot)"]
        RUN["qe_run<br/>(backtest CLI)"]
        FCTR["qe_factor<br/>(factor research CLI)"]
        CREDS["qe_creds<br/>(keyring CLI)"]
    end

    subgraph "Long-running"
        DAEMON["qe_daemon<br/>(live event loop)"]
    end

    subgraph "Core library (linked statically)"
        LIBQE["libqe.a<br/>BarSeries · Engine · DSL · Indicators · Analytics · IO"]
    end

    subgraph "External services"
        IBKR["IBKR Gateway / TWS<br/>(127.0.0.1:7497 / 4002)"]
        ALPACA["Alpaca REST + WS<br/>(api.alpaca.markets)"]
        YAHOO["Yahoo Finance<br/>(quotes + news, free)"]
        KEYCH["OS Keyring<br/>(Keychain / libsecret)"]
    end

    subgraph "On-disk artifacts"
        QECFG[(".qe configs<br/>~/Documents/quant-strategy/")]
        RESULTS[("results.json")]
        FREPORT[("factor_report.json")]
        JOURNAL[("journal.jsonl<br/>~/Library/Application Support/<br/>qe_daemon/state/")]
        LOGS[("daemon logs<br/>~/Library/Application Support/<br/>qe_daemon/logs/")]
    end

    DASH --> LIBQE
    DASH -. quotes and news polls .-> YAHOO
    DASH -. control socket .-> DAEMON
    DASH -. spawns .-> RUN
    DASH -. spawns .-> FCTR
    DASH -. spawns via F6 Deploy .-> DAEMON
    DASH -. reads .-> KEYCH

    RUN --> LIBQE
    RUN --> QECFG
    RUN --> RESULTS

    FCTR --> LIBQE
    FCTR --> QECFG
    FCTR --> FREPORT

    DAEMON --> LIBQE
    DAEMON --> QECFG
    DAEMON --> JOURNAL
    DAEMON --> LOGS
    DAEMON --> IBKR
    DAEMON --> ALPACA
    DAEMON -. reads .-> KEYCH

    CREDS --> KEYCH
```

**Five binaries, one core library.** Every binary links libqe.a
statically — the same backtest engine that powers `qe_run` also
powers `qe_daemon`'s live event loop. The dashboard's
backtest-results renderer and the factor-research viewer read
the same files the CLIs produce. **No client/server inside the
process tree** — the dashboard and the daemon communicate over
a UNIX domain control socket; everything else is a one-shot
subprocess fork (Cmd+S on F3 → `qe_run`).

## Backtest data flow (`qe_run` and the F3 → Cmd+S path)

```mermaid
sequenceDiagram
    participant User
    participant QE_Run as qe_run
    participant Loader as load_qe()
    participant Engine as MultiEngine
    participant Strategy as Strategy (CRTP)
    participant Reporter as HTML / JSON writers

    User->>QE_Run: qe_run config.qe
    QE_Run->>Loader: parse + bind + eval
    Loader-->>QE_Run: BacktestValue<br/>(BoundAst + ExecutionSpec + DataSpec)

    Note over QE_Run: Resolve data spec to BarSeries (yahoo / file / csv_url)

    QE_Run->>Engine: run(MultiBarSeries, ExpressionStrategy, ExecutionConfig)

    loop per bar i = 0 .. N-1
        Engine->>Engine: apply pending fills at bar i (causality)
        Engine->>Strategy: on_bar(i, MultiBarView, MultiPortfolio)
        Strategy->>Strategy: cross_sectional_prepass (if XS sites)
        Strategy->>Strategy: evaluate entry / exit / size
        Strategy-->>Engine: submit_buy / submit_sell
    end

    Engine-->>QE_Run: BacktestResults<br/>(equity, trades, forecasts, attribution)
    QE_Run->>Reporter: write_results_json() + render_html()
    Reporter-->>User: results.json + report.html
```

**Key invariants** (the engine enforces these structurally; no
strategy can bypass them):

- **Causality.** Signals computed at bar `i` may use data from
  bars `0..i`. Fills execute at bar `i+1`'s open, never `i`'s
  close. See `MultiEngine`'s hot loop — `apply_fill` runs BEFORE
  `strategy.on_bar` each tick.
- **Single signal evaluator pool per strategy.** Stateful
  indicators (`sma`, `ema`, `rsi`, `atr`, `rolling_*`, ...) keep
  one in-memory pool per Evaluator. The same Evaluator powers
  backtest AND live — see "Live trading data flow" below.
- **Zero allocations in the hot loop.** Every `std::vector`
  holding trades / equity points is `reserve()`-d to a known
  upper bound. Verified by `tests/test_alloc_counter.cpp`.

## Live trading data flow (`qe_daemon` + dashboard attach)

```mermaid
graph LR
    subgraph "qe_daemon process"
        MAIN[main loop]
        QUOTES[IbkrQuoteSource<br/>tick queue]
        AGG[BarAggregator<br/>per symbol]
        LIVE["LiveEngine<br/>multi-leg + XS barrier"]
        EVAL["Evaluator pool<br/>per leg"]
        RISK["PreTradeRisk<br/>caps + pause + kill"]
        ROUTER["OrderRouter<br/>broker submit"]
        WDOG["DisconnectWatchdog<br/>3-state"]
        JOURNAL["StateJournal<br/>append-only JSONL"]
        SOCK["ControlSocketServer<br/>UNIX domain"]
        HANDLER["DaemonControlHandler<br/>status / orders / log_tail"]
    end

    subgraph "Broker"
        IBKR["IBKR Gateway<br/>(TWS protocol)"]
    end

    subgraph "Dashboard"
        ATTACH[DaemonClient]
        DCACHE[DaemonOrderCache<br/>3s poll]
        F6[F6 TRADE panels]
    end

    IBKR -- ticks --> QUOTES
    QUOTES -- pop --> MAIN
    MAIN --> AGG
    AGG -- closed bars --> LIVE
    LIVE -- evaluate --> EVAL
    EVAL -- intent --> RISK
    RISK -- accepted --> ROUTER
    ROUTER -- orders --> IBKR
    IBKR -- fills --> ROUTER
    ROUTER -- events --> JOURNAL
    JOURNAL -- broadcast --> SOCK

    WDOG -. poll is_connected .-> IBKR
    WDOG -- soft-pause --> RISK
    WDOG -- trip --> RISK

    SOCK <-- control verbs --> ATTACH
    HANDLER --- SOCK
    ATTACH --> DCACHE
    DCACHE --> F6
```

**The dashboard is a polite observer.** It does not own the
broker session — `qe_daemon` does — so closing the dashboard
window leaves trading running unchanged. The dashboard attaches
over the control socket (`~/Library/Application Support/qe_daemon/daemon.sock`),
polls daemon-side state every 3 s, and renders it through the
F6 panels (Working Orders / Executions / Account / Order Log).
Control verbs (`stop`, `kill`, `pause`, `resume`) flow back the
same way.

**Why the daemon is its own process** — broker connections
authenticate per client; one client per account. Putting the
broker session inside the dashboard means closing the dashboard
disconnects the broker → kills the strategy → cancels open
orders. Putting it in a long-running daemon is the only safe
design for headless / overnight strategies.

## Cross-sectional fork-join barrier (EPIC-69)

The live engine's tricky bit. Cross-sectional strategies
(`signalize_universe`, anything with `is_top` / `is_bottom` /
`quantile`) need ALL universe symbols' bars to be fresh before
computing ranks. Without a barrier, the first symbol to close
at session-end would trigger the prepass against 1 fresh +
(N-1) stale slots — the strategy's first rebalance would be
effectively random.

```mermaid
sequenceDiagram
    participant A as Symbol A (1st to close)
    participant B as Symbol B (2nd)
    participant C as Symbol C (3rd, completes quorum)
    participant Eng as LiveEngine
    participant Risk as PreTradeRisk
    participant Router as OrderRouter

    A->>Eng: on_bar at ts_T
    Eng->>Eng: record A in pending set
    Note over Eng: pending 1 of 3 - defer

    B->>Eng: on_bar at ts_T
    Eng->>Eng: record B in pending set
    Note over Eng: pending 2 of 3 - defer

    C->>Eng: on_bar at ts_T
    Eng->>Eng: record C - quorum reached
    Note over Eng: flush_xs_batch with reason quorum

    Eng->>Eng: refresh per_symbol_ctxs (all fresh)
    loop for each XS leg
        Eng->>Eng: cross_sectional_prepass
        Eng->>Eng: evaluate entry / exit
    end
    Eng->>Risk: check intent per fired leg
    Risk-->>Eng: accepted
    Eng->>Router: submit intent
    Note over Eng: pending cleared, ready for next bar
```

**Three flush reasons** appear in the daemon log as
`[xs_batch:reason]` suffixes:

| `reason` | When | Operator action |
|---|---|---|
| `quorum` | Every universe symbol ticked at ts. The healthy case. | None |
| `new_ts` | A bar at a fresh ts arrived while a prior batch was still pending. Prior batch force-flushed with stale slots. | Investigate why a symbol skipped a bar. |
| `timeout` | 5 s wall-clock since batch start without quorum. WARN log. | Same as `new_ts`. |

Non-cross-sectional portfolios bypass the barrier entirely
(per-bar fast path).

## Disconnect watchdog state machine (EPIC-70)

```mermaid
stateDiagram-v2
    [*] --> Healthy
    Healthy --> Healthy: is_connected true
    Healthy --> Paused: 3 misses (soft_pause)
    Paused --> Healthy: reconnect (resume)
    Paused --> Tripped: 30 misses (kill_switch trip)
    Tripped --> [*]: restart required
```

**Transition callbacks** (omitted from the diagram for parser
simplicity):

| Transition | Callback fired |
|---|---|
| `Healthy → Paused` | `on_soft_pause("ibkr_disconnect_soft")` → `PreTradeRisk::set_watchdog_pause` |
| `Paused → Healthy` | `on_resume()` → `PreTradeRisk::clear_watchdog_pause` |
| `Paused → Tripped` | `KillSwitch::trip("ibkr_disconnect")` + cancel every open broker order |

- **Healthy** → normal operation.
- **Paused** → new orders reject (`PreTradeRisk` gate). Open
  orders untouched. Auto-resumes on reconnect within the trip
  window. Reversible.
- **Tripped** → KillSwitch tripped, open orders cancelled,
  daemon needs restart. Sticky.

The dashboard's F6 `broker:` badge reflects the state directly:
`connected` (green) / `paused · Ns` (amber, with miss count) /
`TRIPPED (reason)` (red).

## On-disk layout

```mermaid
graph TB
    subgraph "~/Documents/quant-strategy/"
        WS_BT["backtests/<br/>*.qe (hand-authored)"]
        WS_SW["sweeps/<br/>*.qe (parameter sweeps)"]
        WS_POS["positions/<br/>example_positions.json"]
        WS_OUT["out/<br/>results.json (per-config)"]
    end

    subgraph "~/Library/Application Support/qe_dashboard/"
        DASH_CFG["config.json<br/>(symbols, refresh_seconds,<br/>active_*_path, news_filter_focused, ...)"]
    end

    subgraph "~/Library/Application Support/qe_daemon/"
        DAEMON_SOCK["daemon.sock<br/>(UNIX control socket)"]
        DAEMON_STATE["state/STEM/<br/>journal.jsonl<br/>(STEM = config file stem)"]
        DAEMON_LOGS["logs/<br/>STEM-PID.log"]
    end

    subgraph "~/Library/Keychains/"
        LOGIN_KC["login.keychain-db<br/>alpaca-paper, alpaca-live<br/>(IBKR has no API keys)"]
    end
```

**One folder per concern.** The workspace lives under
`~/Documents/quant-strategy/` (overridable via `Cmd+,` →
Settings → Workspace). The dashboard's own state goes under
`Application Support/qe_dashboard/`. The daemon's runtime state
(socket, journal, logs) lives under
`Application Support/qe_daemon/` — distinct from the dashboard
so multiple daemons (one per `.qe`) can share the same machine.

## Where to go next

- **Build + setup** — [`install.md`](install.md): wizard,
  scripted install, broker credentials.
- **Author a strategy** — [`first-factor-tutorial.md`](first-factor-tutorial.md):
  10-minute walkthrough from clone to a saved `.qe`.
- **Read the `.qe` reference** — [`qe-language.md`](qe-language.md):
  grammar + every builtin.
- **Drive the dashboard** — [`dashboard-walkthrough.md`](dashboard-walkthrough.md):
  per-screen tour.
- **Operate the daemon** —
  [`live-trading-runbook.md`](live-trading-runbook.md) +
  [`qe-daemon-smoke.md`](qe-daemon-smoke.md).
- **Forecasting layer** — [`forecasting.md`](forecasting.md):
  ridge / lasso `predict_return` API.
- **Factor research** — [`factor-research.md`](factor-research.md):
  `qe_factor`-driven IC / long-short flow.
