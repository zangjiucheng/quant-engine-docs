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

`Tab` / `Shift+Tab` cycles all **eight** screens in display order
(F1 MAIN · F2 DEPTH · F3 WKSP · F4 BCKT · F5 RISK · F6 TRADE ·
F7 FCTR · F8 MAP). On F3 WKSP plain `Tab` cycles editor tabs instead,
and `Shift+Tab` remains the escape hatch. F-keys jump directly.
`Cmd+Q` quits. (This used to say "cycles F1–F6" and "F12 quits";
F12 has never been bound.)

Panel focus within a screen is `Ctrl+1..N` and `Ctrl+[` / `Ctrl+]` —
see [Column vocabulary](#6b-column-vocabulary-one-name-per-field) for
what the columns mean once you get there.

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
- ~~WORKING ORDERS at the bottom of this screen mirrors F6's~~
  **Removed in EPIC-47** — F2 has no blotter. Working orders live on
  F6 TRADE only. (Original text kept struck through rather than
  deleted so anyone who remembers the panel learns where it went.)
  The struck line said: WORKING ORDERS at the bottom of this screen mirrors F6's
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
  doesn't yet expose a `cancel_order` control verb. The tooltip
  points at **the broker's own UI** (TWS / Client Portal), which
  is the only place a working order can be pulled. Trip
  kill-switch is *not* an alternative: no kill-switch in this
  codebase cancels a working order — each one only blocks new
  submissions — and the F6 button trips the **dashboard's** switch,
  not the daemon's. See
  [Emergency stop](live-trading-runbook.md#emergency-stop-orders-are-working-and-i-need-them-gone).
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
    - "Trip kill-switch" trips the **dashboard's** switch, in the
      dashboard's process. It does not reach an attached daemon,
      and it cancels nothing at the venue.
- **DAEMON KILL row** (EPIC-88 T88.10, decision D2) — the control
  that reaches the *daemon's* switch, which is the one that governs
  the broker session. Two deliberate clicks inside a 5 s window; the
  arming dies if the state changes under it. Disabled outright when
  the control socket is down, rather than firing into a dead fd and
  reporting success. The row states, permanently and not only after
  you press it, that a kill blocks **new submissions** and does
  **not** cancel working orders.
- **CANCEL ALL row** (EPIC-88 T88.1) — pulls every working order at
  the venue via the daemon's `cancel_all` verb. Also two clicks. It
  runs off the UI thread, so a slow venue does not freeze the
  window. Three things it will not do: report a count it did not
  measure (a book the daemon could not read renders as **UNKNOWN**,
  never as "0 working orders"), fold per-order refusals into a
  total (each one is printed with its own reason), or let you
  believe the book will stay flat — if the kill switch is not
  tripped it says so, because the strategy will submit again on its
  next cycle. **Kill first, then cancel.**

**By design F6 has no order ticket** — new orders are submitted
from F2 DEPTH so you're looking at the book while you size. F6
is for *managing* what's already live + lifecycle (deploy / stop).

### F7 FCTR — read a factor-research report

**Press F7 directly.** It is on the Tab cycle like everything else, but
it is the screen people forget exists, because it is the only one that
renders a *file* rather than live state.

Point it at a `factor_report.json` written by `qe_factor`
(`Cmd+,` → Research → "Factor report path"). Nothing here fetches: the
screen is a reader, and it reloads when the file's mtime changes.

```
FACTORS                      <- cross-factor summary; every factor at once
  FACTOR  IC  IC-IR  t(NW)  SHRP/hp  RET  WINDOWS

<factor selector>            <- then one factor in detail
  IC BY HORIZON              <- one row per horizon
  LONG-SHORT (top vs bottom <quantile>, N-bar rebalance)
  WALK-FORWARD IC            <- rolling windows, if the config asked for them
```

Header line: schema version, the file's mtime **and its age**, and the
date range the study covers. Read the age first. A report that says
`v5 · 2026-07-10 14:22 · 3 weeks ago` is answering a question about a
market that has moved.

**Read `t (NW)`, not `t`.** The plain t-statistic treats an h-bar
forward return sampled every bar as independent and overstates
significance by roughly √h — measured at 2.4–4.5× for h=20 on this
corpus. `t (NW)` is the Newey-West correction. Hover the cell for the
Bartlett lag, the HAC standard error, and the more conservative
non-overlapping cross-check; those four numbers exist so the correction
is auditable rather than magic.

A report older than schema v5 has no NW column. The verdict then falls
back to the uncorrected statistic and **says so on screen with a star**
rather than quietly grading on the wrong number. Re-run `qe_factor` to
get the honest one.

**`WINDOWS` counts by sign.** `12+ 3- / 40` means twelve windows
significantly positive, three significantly negative, out of forty. A
bare count of `|t| > 2` answers "did this factor have an opinion", not
"was it right" — and reading it as the latter is what put a factor
described as "most stable across windows" into a research corpus when a
third of its significant windows pointed the other way.

**`SHRP/hp` is per holding period**, and that basis changed between
report schema v4 and v5: v1–v4 annualised every long-short by 252
regardless of rebalance cadence, inflating it by √rebalance. A v4 report
is marked. F4 BCKT and F5 RISK have columns called `SHARPE/yr`, which
are a different number on a different basis — see
[Column vocabulary](#6b-column-vocabulary-one-name-per-field).

If periods were skipped the panel says how many and what fraction of
the sample actually ran. A study that completed sixty periods and
skipped four hundred is not a study of the period you asked for.

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

- One tile per S&P 500 constituent. **Area** is market cap where
  the universe file has one, and equal-weight for any constituent
  it does not — so a treemap can be a mix of the two. The controls
  row states which rule produced the picture you are looking at
  (`SIZE MKT CAP`, `SIZE EQUAL`, or `SIZE MKT CAP · N of M have
  none`), because a $3 T tile sitting next to a fallback tile is
  otherwise a picture of nothing. **Color** is the % change over
  the selected window, clamped at ±3 %, finviz-style red ↔ grey ↔
  green.
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
| `Cmd+Shift+X`                    | Trips **the dashboard's own** kill-switch (not the daemon's — they are separate objects in separate processes). To reach an attached daemon's switch use the F6 **DAEMON KILL** row (EPIC-88 T88.10); this chord deliberately has no daemon equivalent, because a one-way remote kill should not be a keystroke. New `submit_order` is rejected. **Orders already working at the venue are untouched** — nothing here cancels them; cancel + read-only calls stay open precisely so you can go drain them yourself. **First-trip-wins**: only the first chord captures a reason; later presses are no-ops. Persists until restart. See [Emergency stop](live-trading-runbook.md#emergency-stop-orders-are-working-and-i-need-them-gone). |
| Actual positions ≠ broker positions | Reconcile worker pops a **read-only** drift modal. No auto-fix. You decide whether to adjust manually in the broker UI. See [`live-trading-safety.md`](live-trading-safety.md). |
| Broker socket drops              | Top-bar badge → `BROKER OFFLINE`. Reconnect via Broker menu or F6 SAFETY `[Reconnect broker]`. |
| Day-loss limit exceeded          | **In the daemon (EPIC-88 T88.4): evaluated continuously**, on every mark update, against the *strategy's* marked-to-market P&L anchored on `execution(capital = ...)`. Two consecutive breaching marks trip the kill-switch — so a position can no longer bleed through the limit all session just because nothing was being submitted. New submissions then stop; working orders are **not** cancelled. **In the standalone dashboard**, SafeBroker's per-submit check is still the only gate: it rejects that one order and trips nothing. IBKR `day_pl` arrives via the account stream. |
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

## 6a. The command line

`/` or `Cmd+K` focuses it. `Esc` clears the line and gives the keyboard
back.

```
[SECURITY] [CODE [SUB] [ARG]]
```

- `AAPL` — focus that symbol, stay where you are
- `AAPL DEPTH` — focus it and jump to F2
- `RISK GRK` — F5 RISK, Greeks tab
- `BCKT out/run.json` — F4 BCKT re-pointed at a results file
- `FCTR report.json` — F7 FCTR re-pointed at a factor report
- `=MAP` — the **ticker** MAP, not the screen. `=` escapes a collision
- `KILL` — trips this process's kill switch (same as `Cmd+Shift+X`)

`Tab` opens the completion list, `↑` / `↓` step it (or step history when
no list is showing), `Enter` applies the selection or runs the line.

**The authoritative list of codes is `HELP`, typed into the bar itself.**
It is generated from the same mnemonic table the parser reads
(`kDefaultMnemonics`), so it cannot disagree with what the app accepts.

That is why there is no table of codes in this document. T87.16 removed
a hand-typed copy of the F5 tab list from the mnemonic descriptions for
exactly this reason — it printed the tabs twice, once generated and once
by hand, and the hand-typed one was already drifting. A second copy
here, updated by whoever remembers, would be the same mistake at a
larger distance from the code.

## 6b. Column vocabulary — one name per field

Every screen is a grid of tables, and until EPIC-87 T87.31 the same
field wore a different name on each one. Executions were the clearest
case: F1 TAPE, F4 EXECUTION and F6 EXECUTIONS render the same shape of
row — a fill — and shared not one column name between them.

This is the canonical list. **A new column takes a name from this table
or adds a row to it.** Two names for one field is a bug in a terminal:
it is the mechanism by which an operator reads a number as the wrong
quantity.

| Field | Canonical | Format | Align | Where the unit lives |
|---|---|---|---|---|
| Instrument | `SYM` | ticker, upper | left | — |
| Traded price of an event | `PX` | 2 dp | right | header currency |
| Last traded price | `LAST` | 2 dp | right | header currency |
| Valuation price | `MARK` | 2 dp | right | header currency |
| Your quantity | `QTY` | integer, or 2 dp if fractional | right | shares/contracts by context |
| Resting book depth | `SIZE` | integer | right | shares |
| Cash value of a fill | `NOTIONAL` | compact (`1.2M`) | right | account currency |
| Cash value of a position | `MKT VAL` | compact | right | account currency |
| Unrealised P&L | `OPEN P&L` | signed compact | right | account currency |
| Realised P&L | `REALIZED` | signed compact | right | account currency |
| Session P&L | `DAY P&L` | signed compact | right | account currency |
| Return, fractional | `RET%` | signed, 2 dp, `%` | right | in the label |
| Wall-clock instant | `TIME` | `hh:mm:ss` local | left | date via separator row |
| Elapsed duration | `AGE` | `4m`, `2h` | right | in the value |
| Weight of a leg | `WGT%` | signed, 2 dp, `%` | right | in the label |

### Names that look like duplicates and are not

- **`LAST` / `MARK` / `PX`** — a trade print, a valuation, and the price
  of one specific event. Collapsing them would make "the price" mean
  three things.
- **`QTY` / `SIZE`** — yours versus the book's. A ladder's `SIZE` is
  resting liquidity you do not own.
- **`NOTIONAL` / `MKT VAL`** — what a fill cost versus what a holding is
  worth now.
- **`TIME` / `AGE`** — an instant versus a duration. A column that
  sometimes shows one and sometimes the other is the reason this
  distinction is written down.
- **`SHARPE/yr` / `SHRP/hp`** — F4 BCKT and F5 RISK annualise per year;
  F7 FCTR's long-short is per holding period, and that basis *changed*
  between report schema v4 and v5. **The basis belongs in the label**,
  not in a doc the reader does not have open. Any future Sharpe column
  carries its basis the same way.

### Renamed by T87.31

`SYMBOL`→`SYM`, `PRICE`→`PX`, `WHEN`→`TIME`, `VALUE`→`NOTIONAL`,
`uP&L`→`OPEN P&L`, `RETURN`→`RET%`, `SHARPE`→`SHARPE/yr`.

## 7. Where to look next

- Sweep workflow & cell-click reruns: [`docs/qe-language.md`](qe-language.md) §sweep
- Multi-strategy portfolios: [`docs/multi-strategy.md`](multi-strategy.md)
- Walk-forward harness: [`docs/walk-forward.md`](walk-forward.md)
- Options pricing model: [`docs/options-model.md`](options-model.md)
- TWS protocol details (for hacking on `ibkr_connection.cpp`): [`docs/ibkr-tws-protocol.md`](ibkr-tws-protocol.md)
