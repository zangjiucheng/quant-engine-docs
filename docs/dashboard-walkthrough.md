# Dashboard walkthrough — from cold start to first paper fill

End-to-end tour of `QE Dashboard.app`: setup → research →
backtest → live paper trading. Read this once; come back to
the per-feature docs for depth.

Cross-references:

- Connectivity: [`docs/ibkr-connectivity.md`](ibkr-connectivity.md)
- Safety model: [`docs/live-trading-safety.md`](live-trading-safety.md)
- Strategy DSL: [`docs/qe-language.md`](qe-language.md)
- First-IBKR-paper checklist: [`docs/ibkr-paper-verification.md`](ibkr-paper-verification.md)

## 0. One-time setup

### Build

```bash
cmake --preset=release
cmake --build --preset=release -j
open "build/release/bin/QE Dashboard.app"          # macOS
./build/release/bin/qe_dashboard                   # Linux
```

First launch writes a workspace at
`~/Library/Application Support/qe_dashboard/workspace/` (macOS)
or the XDG equivalent on Linux: six starter `example_*.qe`
files plus a `positions.json`. F3 WKSP opens to that folder so
you're never staring at an empty pane.

### Pick a broker

Two paths, both safe to learn on. Configure once, change later
via `Cmd+,`.

| Broker        | Cred source                                     | Notes |
| ------------- | ----------------------------------------------- | ----- |
| `alpaca-paper`| `ALPACA_KEY_ID` / `ALPACA_SECRET_KEY` env vars  | US equities only. No setup beyond the env vars. |
| `ibkr-paper`  | TWS or IB Gateway logged into a paper account   | Multi-asset, multi-currency. Need Gateway running first — see [`ibkr-connectivity.md`](ibkr-connectivity.md). |
| `ibkr-live`   | TWS/Gateway logged into a live account          | Requires `IBKR_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING` env var **and** a typed-confirmation modal. Don't pick this until you've done the paper checklist end-to-end. |

## 1. `Cmd+,` — Settings pass

Open the Settings modal, walk top-to-bottom:

| Section       | Field                | What to set                          |
| ------------- | -------------------- | ------------------------------------ |
| General       | Refresh interval     | 5s default is fine                   |
| General       | Vim mode             | On if you want vim motions in F3     |
| Data feeds    | Yahoo news           | On — drives F1 NEWS                  |
| Data feeds    | L2 streaming         | On — drives F2 DEPTH (needs Alpaca creds) |
| Trading       | Broker               | `alpaca-paper` or `ibkr-paper`       |
| Trading       | IBKR host / port     | `127.0.0.1` / `4002` for Gateway paper |
| Trading       | IBKR client id       | `1` (any unused int)                 |
| Research      | Factor window / K    | Defaults are fine                    |

Close Settings → top bar shows the broker badge (`PAPER ·
alpaca-paper`, `PAPER · ibkr-paper`, or `LIVE` in red). Edits
auto-save on close and on app exit.

The watchlist is **not** edited from Settings — edit it
in-place on F1 MAIN (Cmd+P style fuzzy add, ↑/↓ to focus).

## 2. The screens, in workflow order

`Tab` / `Shift+Tab` cycles through F1–F6 in display order. F-keys
jump directly. F12 quits, `Cmd+Q` also quits.

### F1 MAIN — see the market

Watchlist + ImPlot candles + positions/tape + Yahoo news.

- `↑` / `↓` (or `j` / `k` in vim mode) — focus prev/next symbol
- Click a watchlist row — focus that symbol (chart panel follows)
- `+ ADD` — async fuzzy search by ticker **or** company name
- POSITIONS rows are LIVE from `positions_json`; TAPE shows the
  most recent fills your broker reported
- **NEWS panel** — Yahoo headlines for every watched symbol on
  a 60 s refresh. The `[all]` / `[focus]` chip under the panel
  header (EPIC-71) toggles between every cached headline and
  only headlines whose `related_tickers` includes the focused
  symbol (the one highlighted in the watchlist). The cache keeps
  sweeping the full watchlist regardless — swapping focus
  refreshes the filtered view instantly and the focused symbol's
  coverage stays as fresh as everything else. State persists in
  the dashboard config (`news_filter_focused`).

### F2 DEPTH — drill into microstructure

L2 book + Time & Sales + spread/imbalance/vol-burst widgets +
ORDER TICKET.

- Use this to **time an entry** — look at where the liquidity
  is, what's clearing, what the imbalance is doing.
- ORDER TICKET is where new orders get submitted (qty, side,
  type, limit price → `Submit`). Read-only when `broker = Disabled`.
- WORKING ORDERS at the bottom of this screen mirrors F6's
  blotter — convenient while you're staring at the book.

### F3 WKSP — write / edit strategies

File tree (left) + multi-tab editor (right) + run log (bottom).

- Click any `.qe` file in the tree to open as a tab.
- `Shift+H` / `Shift+L` cycle tabs; `Cmd+W` closes the active tab.
- **`Cmd+S`** is the bus stop:
  - `.qe` backtest → forks `qe_run`, stdout/stderr into the log,
    `results.json` lands → F4 BCKT auto-refreshes.
  - `.qe` sweep → loads spec into F4 BCKT's sweep panel, runs it.
  - `positions.json` → re-mounts as live positions, F1 / F5 auto-refresh.
- Vim mode (toggle `Cmd+Shift+V`): `i a I A o O` insert,
  `h j k l w b e 0 ^ $ gg G` motions with counts, `x X dd yy p P
  u Ctrl-R` edits, `D C v V` line/visual operations, `:w :bd :q` ex.

Minimal `.qe` example (in `backtests/example_ma_spy.qe`):

```
let fast = 10
let slow = 50

backtest(
  data     = yahoo("SPY", "1d", "2024-01-01", "2024-12-31"),
  strategy = signal(
    entry  = cross_above(sma(close, fast), sma(close, slow)),
    exit   = cross_below(sma(close, fast), sma(close, slow)),
    symbol = "SPY",
    size   = 1.0,
  ),
  execution = execution(capital = 100_000, commission_bps = 1.0),
  output    = output(results = "out/ma_spy.json"),
)
```

Full grammar: [`docs/qe-language.md`](qe-language.md).

### F4 BCKT — review results

Sweep matrix (top) + equity curve + KPIs + recent fills +
monthly returns heatmap.

- If F3 saved a single-config backtest: equity / KPIs / monthly
  matrix render off the produced `results.json`.
- If F3 saved a sweep: sweep matrix on top (2D bucketed, ◆ on
  the best cell). **Click any cell** → re-runs that config with
  let-overrides, re-points `results_json` at the new file,
  rest of the screen swaps to that cell's results.
- Monthly matrix uses a 3-tier diverging palette (red /
  neutral / green); NaN cells are dim gray.

### F5 RISK — portfolio risk

Always-on meta strip (NAV / Gross / Net / Leverage / Long-Short /
VaR95 / ES99) over four sub-tabs:

| Tab          | What's live                                                   |
| ------------ | ------------------------------------------------------------- |
| Overview     | Concentration glance (per-symbol or per-sector if mapped)     |
| Greeks       | Black-Scholes Δ/Γ/ν/Θ/ρ per leg + Σ; needs `positions_json.options[]`. Equity-only Δ$ if no options. |
| VaR & Stress | VaR/ES via historical / parametric + 8 spot+vol scenarios     |
| Factor       | PCA correlation matrix + portfolio exposures + per-PC stats + strategy IC |

Each tab degrades to a "no data — set `positions_json`" hint
when its inputs aren't available. Active tab is persisted
across launches.

### F6 TRADE — manage live orders + deploy daemons

F6 has two layouts depending on whether `qe_daemon` is attached:

**No daemon attached** (Deploy panel takes the top half):

```
DEPLOY  ·  start qe_daemon on a live(...) .qe (F3 → Cmd+S → switch here)
─────────────────────────────────────────────────────────────────────
File:     ~/Documents/quant-strategy/deploy_it_long_only_1000_paper.qe
Broker:   ibkr-paper  [PAPER]
IBKR:     127.0.0.1:4002  client_id=1
Symbols:  30 (AAPL, ...)
Capital:  $1000

[Arm deploy]            (two-click guard; live brokers also gate
                         on typing the broker name into a confirm box)

ACCOUNT · USD                           ORDER LOG
... empty until a daemon attaches ...   ... empty ...

SAFETY · kill: armed · reconcile: — · broker_session: OFFLINE
       · daemon: — · broker: —
[Trip kill-switch]   [Reconnect broker]   [Re-reconcile now]
[Stop daemon]
```

**Daemon attached** (normal 4-panel layout, reading from the
daemon's control socket):

```
WORKING ORDERS                          EXECUTIONS (last 50)
ID    SYM   SIDE QTY    PX     STATE    TIME      SYM  SIDE QTY  PX
o-17  AAPL  BUY  0/10   150.50 WORK     14:32:05  SPY  BUY  1    495.10
o-18  SPY   SELL 3/5    495.50 PART     14:31:48  AAPL SELL 5    149.80
[Cancel] disabled in daemon mode        …
STAGED ORDERS          (only for staged-entry strategies)
AAPL  BUY   1/2   5    PENDING  Tue 09:30:30

ACCOUNT · USD                           ORDER LOG
cash          100,000.00                14:30:01 submit_attempt  AAPL BUY 10 @ 150.50
buying power  400,000.00                14:30:02 submit_accepted AAPL
equity        100,231.12                14:31:48 fill            SPY 1 @ 495.10
day P&L       +231.12 (+0.40%)          14:32:05 fill            AAPL 5 @ 150.42
  account: DUQ526944                    … auto-stick to bottom

SAFETY · kill: armed · reconcile: clean · broker_session: PAPER ibkr-paper
       · daemon: attached · broker: connected
EVAL   · last 2026-06-04 20:00Z ok 3 orders  ·  2 alerts
[Trip kill-switch]   [Reconnect broker]   [Re-reconcile now]
[Stop daemon]
```

- **Deploy panel** (EPIC-66) — visible only when no daemon is
  attached. Reads `cfg.active_live_path` (set by F3 Cmd+S on a
  `live(...)` file), previews the file, two-click guard for
  paper brokers + typed broker-name confirm for live. Spawns
  `qe_daemon` via double-fork + setsid so the daemon survives
  the dashboard exiting; redirects stdio to
  `~/Library/Application Support/qe_daemon/logs/<stem>-<pid>.log`.
  Polls `kill(pid, 0)` each frame after spawn and tails the
  log's `[error]` lines if the daemon dies pre-attach so you
  see the failure inline instead of digging through logs.
- WORKING ORDERS / EXECUTIONS / ACCOUNT / ORDER LOG (EPIC-67) —
  when the daemon is attached, all four read from
  `DaemonOrderCache` which polls the daemon's `orders` /
  `positions` / `equity` / `log_tail` control verbs every 3 s.
  Same `OrderSnapshot` shape as the local-broker path, so the
  render code is uniform.
- Cancel button is **disabled in daemon mode** — the daemon
  doesn't yet expose a `cancel_order` control verb. Tooltip
  points at Trip kill-switch as the working alternative
  (cancels ALL open orders).
- **STAGED ORDERS sub-panel** (EPIC-74) — appears under WORKING
  ORDERS only when the deployed strategy uses `staged_entry` /
  `staged_exit` and at least one slice is pending or recently
  fired. One row per slice: symbol, side, `n/N` fired, qty,
  state (pending / fired / cancelled), next fire time. See
  [Staged entry / exit](staged-entry.md).
- **EVAL line** (EPIC-75) — one inline row in the SAFETY footer:
  `EVAL · last <close> ok 3 orders · N alerts`. Green ok /
  yellow empty / red error; the alerts chip turns red on any
  `eval_missed` / `eval_replay_mismatch` /
  `eval_idempotency_violation`. Hidden until the daemon has eval
  activity. See [Eval self-healing](eval-self-heal.md) —
  including same-evening recovery via `qe_daemon backfill`.
- **Resizable panels** (EPIC-82) — the 2×2 grid has a draggable
  vertical splitter between the columns (shared across both rows)
  and a horizontal one between the rows. Positions persist in
  `config.json` (`layout.trade.left_col_w` / `top_row_h`). The
  SAFETY footer keeps a fixed height.
- SAFETY footer badges:
    - **`kill:`** — KillSwitch state. `armed` (green) /
      `TRIPPED (reason)` (red).
    - **`reconcile:`** — reconcile-worker state if the dashboard
      owns the broker; `—` otherwise.
    - **`broker_session:`** — dashboard's own broker session
      (legacy / pre-daemon mode).
    - **`daemon:`** — control-socket attach state. `attached`
      (green) means the dashboard can read state + send commands.
    - **`broker:`** — daemon's view of its broker link
      (EPIC-70). `connected` (green) / `paused · Ns` (amber, with
      miss count — soft-pause, reversible on reconnect) /
      `TRIPPED (reason)` (red, hard trip).
- SAFETY footer buttons: Trip kill-switch · Reconnect broker ·
  Re-reconcile now · **Stop daemon** (EPIC-66 — sends `stop`
  control verb for a graceful shutdown).

**By design F6 has no order ticket** — new orders are submitted
from F2 DEPTH so you're looking at the book while you size. F6
is for *managing* what's already live + lifecycle (deploy / stop).

### F8 MAP — sector-grouped market mood

```
WINDOW [1D] [5D] [1M] [3M] [YTD]   REFRESH   LAST 14:31:48 · 491/503 OK
┌──────────────────────────────┬──────────────────────────┬────────┐
│  Information Technology      │  Financials              │ Energy │
│  ┌─────┬─────┬───┬───┬───┐   │  ┌────┬───┬──┬──┬──┐    │  ┌──┐  │
│  │AAPL │MSFT │NVDA│GOO│META│  │  │JPM │BAC│..│..│..│    │  │XOM│  │
│  ├─────┼─────┼───┴───┴───┤   │  ├────┴───┴──┴──┴──┘    │  ├──┤  │
│  │AMZN │CRM  │ ... 50+   │   │  │  ... 70+ tiles ...   │  │CVX│  │
│  └─────┴─────┴───────────┘   │  └────────────────────────┘  └──┘  │
├──────────────────────────────┴──────────────────────────┴────────┤
│  ... 8 more sectors ...                                          │
└──────────────────────────────────────────────────────────────────┘
```

- One tile per S&P 500 constituent. **Area** is equal-weight in
  v1 (every ticker the same size; sector area = member count).
  **Color** is the % change over the selected window, clamped at
  ±3 %, finviz-style red ↔ grey ↔ green.
- **Hover** a tile for ticker / name / sector / industry / window
  % / last + prev close.
- **Right-click** copies the ticker to the clipboard — paste
  straight into F1 watchlist or an F3 WKSP `.qe` config.
- **REFRESH** forces an immediate sweep; the cache otherwise
  refreshes every 30 s.
- Universe lives at `data/universe/sp500_gics.csv` (regenerated
  by `tools/gen_sp500_universe.py` from Wikipedia). If the file
  is missing, the panel renders an empty-state hint and the rest
  of the dashboard is unaffected.

## 3. End-to-end paper-trade flow

```text
0. Gateway / Alpaca creds OK     · top-bar badge shows PAPER
1. F1 MAIN  : add symbol         · ↑/↓ to focus the one you want
2. F2 DEPTH : eyeball the book   · pick a limit price
3. F2 ticket: Submit             · WORKING ORDERS row appears
4. F6 TRADE : watch the fill     · ORDER LOG / EXECUTIONS update
5. (repeat 2–4 as needed)
6. Cmd+Shift+X if anything looks wrong  → KILL ·  no new orders
7. Cmd+Q
```

Research-only session (no live broker):

```text
1. F3 WKSP : edit a .qe          · vim or plain text
2. Cmd+S   : qe_run forks        · log streams into F3
3. F4 BCKT : equity / KPIs       · auto-refreshes on save
4. (iterate)
5. Cmd+Q
```

## 4. Safety net — what each guard does

| Trigger                          | Effect                                                                                    |
| -------------------------------- | ----------------------------------------------------------------------------------------- |
| `Cmd+Shift+X`                    | Global kill-switch tripped. New `submit_order` is rejected. Cancel + read-only calls keep working so you can drain open orders. **First-trip-wins**: only the first chord captures a reason; later presses are no-ops. Persists until restart. |
| Actual positions ≠ broker positions | Reconcile worker pops a **read-only** drift modal. No auto-fix. You decide whether to adjust manually in the broker UI. See [`live-trading-safety.md`](live-trading-safety.md). |
| Broker socket drops              | Top-bar badge → `BROKER OFFLINE`. Reconnect via Broker menu or F6 SAFETY `[Reconnect broker]`. |
| Day-loss limit exceeded          | SafeBroker rejects new orders once `day_pl` crosses the configured threshold. IBKR `day_pl` arrives via the account stream. |
| `ibkr-live` selection            | Requires `IBKR_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING` env var **and** typing the broker name into the confirmation modal. Triple gate; deliberately annoying. |

## 5. Persistence — what survives a restart

| State                          | Persisted? | Where                                |
| ------------------------------ | ---------- | ------------------------------------ |
| Watchlist / focused symbol     | ✓          | `config.json` (auto-save on exit)    |
| Broker selection + IBKR knobs  | ✓          | `config.json`                        |
| Workspace path + active backtest/sweep file | ✓ | `config.json`                  |
| Active F5 RISK tab             | ✓          | `config.json`                        |
| Layout splitter positions      | ✓          | `config.json` (per-screen)           |
| F3 editor open tabs            | ✗          | Re-open from the tree next session   |
| Working orders at the broker   | ✓ (broker side) | TWS / Gateway / Alpaca keeps them; dashboard re-syncs on reconnect |
| Kill-switch state              | ✗          | Always armed on launch — by design   |

## 6. Common surprises

- **F6 ACCOUNT shows zeros under IBKR.** Older builds returned
  an empty `AccountInfo` stub. Update to a recent build with the
  account-stream wiring; cash / buying_power / equity / day_pl
  populate from `REQ_ACCOUNT_DATA`.
- **Dashboard launches with `BROKER OFFLINE` even though Gateway
  is up.** Expected for a second or two: the IBKR broker now
  constructs on a background thread (EPIC-81) so the window
  appears immediately instead of blocking on `::connect()` —
  which used to look like a ~75 s freeze when the Gateway was
  *down*. The badge flips to `READY` when the connect resolves;
  if it stays `OFFLINE`, check the Gateway and the log.
- **`Historical Data Farm is Inactive: ushmds` in Gateway.**
  Informational, not an error. Means "no one's requesting
  historical bars; the connection is dormant". The dashboard
  itself doesn't use IBKR historical data (Yahoo handles its
  charts); `qe_daemon` only requests it once per deploy for
  warmup. Ignore.
- **F4 BCKT empty after `Cmd+S`.** The `.qe`'s `output(results = …)`
  path is relative to the workspace root; check the path printed
  in F3's log pane resolves to a writable directory.
- **Vim cursor not highlighted on F3 tree.** Already fixed in a
  recent build; if you see this, rebuild.
- **F6 TRADE blotter row says `WORK` but never moves.** Check
  the order is within trading hours for the symbol's exchange.
  Outside hours, IBKR holds the order; Alpaca paper accepts
  but won't simulate fills.

## 7. Where to look next

- Sweep workflow & cell-click reruns: [`docs/qe-language.md`](qe-language.md) §sweep
- Multi-strategy portfolios: [`docs/multi-strategy.md`](multi-strategy.md)
- Walk-forward harness: [`docs/walk-forward.md`](walk-forward.md)
- Options pricing model: [`docs/options-model.md`](options-model.md)
- TWS protocol details (for hacking on `ibkr_connection.cpp`): [`docs/ibkr-tws-protocol.md`](ibkr-tws-protocol.md)
