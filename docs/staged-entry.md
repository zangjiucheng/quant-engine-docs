# Staged entry / staged exit

When a backtest tells you to buy 100 shares of AAPL at Monday's open,
the cleanest implementation is a single market order at 09:30:30 ET.
For a $10k+ NAV with 5-10 positions that single print can be 10-50 bps
worse than the day's VWAP — random tape noise that didn't exist in the
backtest. **Staged entry** spreads the same intent across 2-4 windows
over 1-2 trading days so the basket's entry cost averages across
multiple prints.

The same machinery works for exits (shipped as `staged_exit`).

## When to use it

Recommended threshold: **$10k+ NAV with positions large enough that a
single slice still represents < 30 bps of impact**. Below that, the
extra commission per slice + the schedule complexity outweighs the
diversification benefit. Above it, you cut single-print risk roughly
in half.

Don't use staged entry for:

- Strategies with one-shot fast triggers (mean-revert intraday).
  Staging adds latency that destroys the edge.
- Sweep / walk-forward configs that you want to compare bit-exact
  against earlier runs — staging changes the fill sequence.
- Crypto / 24h venues. The DSL window types (`"open"` / `"close"`)
  are NYSE-session-specific in v1.

## The canonical 30 / 30 / 25 / 15 split

```python
backtest(
  data = yahoo("AAPL", "1d", "2026-01-01", "2026-12-31"),
  strategy = signal(
    entry = cross_above(sma(close, 10), sma(close, 50)),
    exit  = cross_below(sma(close, 10), sma(close, 50)),
  ),
  execution = execution(
    capital = 100_000,
    commission_bps = 1.0,
    slippage_bps = 0.5,
    entry_schedule = staged_entry(
      slices = [
        slice(day_offset = 0, window = "open",  pct = 0.30),
        slice(day_offset = 0, window = "close", pct = 0.30),
        slice(day_offset = 1, window = "open",  pct = 0.25),
        slice(day_offset = 1, window = "close", pct = 0.15),
      ]),
  ),
)
```

The slice list is evaluated **once per strategy signal**. When the
strategy decides to buy 100 AAPL on Monday at the close, the engine
produces four `ScheduledOrder` rows:

| Slice | Day | Window | Qty |
|------:|:----|:-------|----:|
| 1 | Mon | open  | 30 |
| 2 | Mon | close | 30 |
| 3 | Tue | open  | 25 |
| 4 | Tue | close | 15 |

The same expansion runs in **both** backtest and live, so walk-forward
PnL reflects what the daemon will actually execute.

## Function reference

### `slice(day_offset, window, pct, limit_offset_bps = 0)`

One row in a schedule. All kwargs except `limit_offset_bps` are required.

- **`day_offset`** (non-negative integer) — `0` = first session at or
  after the signal, `1` = the session after that, etc. Holidays and
  weekends are skipped by the calendar walk, so `day_offset = 1` from a
  Friday signal lands on Monday.
- **`window`** (`"open"` | `"close"`) — anchor within the session.
  - `"open"` fires at `session_open + 30s` (configurable via
    `open_offset_s`).
  - `"close"` fires at `session_close - 5min` (configurable via
    `close_offset_s`).
- **`pct`** (`0.0`-`1.0`) — fraction of the parent target qty this
  slice takes. The `staged_entry` / `staged_exit` builtin verifies
  the slice list sums to `1.0 ± 1e-6`.

  **Small-qty rounding caveat.** Each slice's nominal qty
  (`target * pct`) is rounded to whole shares via `std::round` so
  the broker accepts the order, and the **last** slice in the list
  absorbs the residual exactly so the sum equals the parent intent.
  When the parent target is small enough that `round(target * pct)`
  flattens to `0` or `target` on any non-final slice, that slice is
  skipped — and the schedule looks like it "did nothing" at the
  trade-log level. Examples for a 50 / 50 two-slice split:

  | Parent target | Slice 0 (round 50%) | Slice 1 (residual) |
  |--:|--:|--:|
  | `1` | `1` (round(0.5)=1) | `0` (skipped) |
  | `2` | `1` | `1` |
  | `0.5` | `0` (skipped, round(0.25)=0) | `0.5` |
  | `100` | `50` | `50` |

  So a portfolio backtest where each leg's qty is ~1 share will
  produce *identical* trades with or without a 50 / 50 schedule.
  This is by design (whole-share broker compat + exact residual
  sum) — not a bug in the executor. If you need to measure slice
  diagnostics under small per-leg capital, scale the strategy's
  `size` up so each slice rounds to ≥ 1 share, or pin the qty
  upstream via a custom size expression.
- **`limit_offset_bps`** (integer, default `0`) — reserved. v1 always
  fires market orders; passive-limit slices land in a follow-up EPIC.

### `staged_entry(slices, on_partial_fill?, on_window_miss?, open_offset_s?, close_offset_s?)`

Wraps a slice list with policy knobs. Exists as `staged_exit` under the
same shape — naming is for readability; the engine routes intents to
`entry_schedule` for buys and `exit_schedule` for sells regardless.

- **`slices`** (required, non-empty array of `slice(...)` values).
- **`on_partial_fill`** (`"continue"` | `"halt"`, default `"continue"`)
  — what to do when a prior slice's broker fill was less than its
  scheduled qty. `"continue"` fires the next slice unchanged;
  `"halt"` cancels all remaining slices and journals a notice.
- **`on_window_miss`** (`"skip"` | `"reschedule_next"`, default
  `"skip"`) — what to do when a slice's `fire_at_ns` has already
  passed by the time the dispatcher checks. `reschedule_next` is
  reserved for v2.
- **`open_offset_s`** (non-negative integer, default `30`) — seconds
  after session open for `"open"` slices.
- **`close_offset_s`** (non-negative integer, default `300`) — seconds
  before session close for `"close"` slices.

## Backtest vs live semantics

Both paths call the same `SliceSchedule::expand()` and produce
identical `ScheduledOrder` lists. They differ only in **when** they
dispatch:

| | Backtest | Live |
|-|-|-|
| Drain trigger | `bar.ts >= fire_at_ns` (per bar) | wall-clock `now >= fire_at_ns` (per 1Hz tick) |
| Where fills land | `pf.apply_fill` at the next bar's open/close per `fill_model` | `IOrderRouter::submit` → broker NBBO + slippage |
| Restart recovery | irrelevant | journal replay rebuilds Pending queue |

For daily bars timestamped at session open (Yahoo style), multiple
intra-session slices may collapse into one bar's fill — e.g. the
30 / 30 / 25 / 15 split fires as `bar+1: 60 shares, bar+2: 40 shares`
because day-0 open and day-0 close both resolve to a `fire_at_ns`
before bar+1 closes. With intraday minute bars the resolution is fine
enough that each slice fires at its intended window.

## Restart recovery

The daemon journals every state transition:

- `slice_scheduled` — one row per slice on `schedule_intent`.
- `slice_fired` — when the dispatcher submits to the broker.
- `slice_cancelled` — kill-switch trip, operator cancel, or
  window-miss skip.

On startup, `OrderScheduler::hydrate_from_journal` replays these rows
in order. Latest-wins per `(parent_signal_id, slice_idx)` collapses
mid-flight crashes: a slice that fired right before the daemon
crashed comes back as `Fired` (not re-submitted on resume), and a
slice that was queued but never fired comes back as `Pending` (will
fire on the next tick after its `fire_at_ns`).

## F6 TRADE STAGED panel

When a daemon has a scheduler attached and at least one pending /
recently-fired slice exists, the F6 TRADE WORKING ORDERS panel grows
a `STAGED ORDERS` subsection:

```
STAGED ORDERS
─────────────────────────────────────────────────────────────
SYMBOL  SIDE  SLICE  QTY   STATE     FIRE TIME
AAPL    BUY   3/4    25    pending   06-09  09:30:30
MSFT    SELL  1/2    100   fired     06-08  15:55:00
NVDA    BUY   2/4    30    cancel    06-08  19:55:00
```

Status colors: `pending` is green (`theme::Ok`), `fired` is dim
(`theme::Dim`), `cancel` is red (`theme::Bad`).

The panel polls the daemon's `scheduled_orders` control verb on the
standard `DaemonOrderCache` cadence — 3 s during the session, backs
off to 60 s in off-hours per EPIC-72.

## What's out of scope (v1)

- **Adaptive sizing** — skipping a slice if VWAP > threshold, scaling
  the next slice if slice 1 filled badly. Smart routing; separate EPIC.
- **Intra-window algorithms** — VWAP / TWAP inside one slice. Venue
  primitive, not slicing.
- **Cross-symbol coordination** — "fill all 5 longs in the same
  window before any of the next-window slice fires". Basket-level
  state, defer.
- **Passive limit slices** — `limit_offset_bps` is parsed and stored
  but the live dispatcher always fires market orders.
- **Adjust partial-fill policy** — re-spreading a partial residual
  over remaining slices. Interacts awkwardly with concurrent broker
  fills; v2.
