# Your first factor — a 10-minute tour

This walks the closed loop from a blank workspace tab to a
**walk-forward IC** verdict on a real factor. We use SPDR sector
ETFs because their cross-section actually has a non-trivial signal
— good for "did anything happen?" sanity checks. Everything below
runs inside the dashboard; no terminal, no separate process.

If you want the *why* of each step (what IC means, how the
long-short is built, what the t-stat threshold is), the reference
is [`factor-research.md`](factor-research.md). This page is the
how — narrow on purpose.

## Prerequisites

- `qe_dashboard` builds and launches.
- On first launch it auto-bootstraps a workspace with
  `example_*.qe` starters; we'll create a new file next to them.

## 1. Open WKSP and create the file

Hit **F3** to open WKSP. In the left pane, right-click
`backtests/research/` → **New file…** → name it
`first_factor.qe`. (No research/ folder? Right-click
`backtests/` → **New section…** → `research`, then add the file
inside.) A blank editor tab opens.

> **What you should see:** an empty editor on the right, your
> new filename highlighted in the tree on the left.

## 2. Paste a minimal `research(...)`

```qe
# first_factor.qe — momentum on SPDR sectors.
research(
  universe = universe(
    symbols = ["XLK", "XLF", "XLE", "XLV", "XLY",
               "XLP", "XLI", "XLB", "XLU"],
    data    = yahoo_template("1d", "2018-01-01", "2025-01-01"),
  ),
  factors = [
    factor("momentum_20", expr = (close / sma(close, 20)) - 1.0),
  ],
  horizons  = [1, 5, 20],
  quantiles = 3,
  rebalance = 5,
  output    = output(report = "out/first_factor_report.json"),
)
```

Nine symbols, one factor (20-bar price momentum), three
horizons. `quantiles = 3` splits the nine names into thirds for
the long-short.

## 3. Cmd+S

WKSP detects the file is a `research(...)` value and forks
**`qe_factor`** on it (not `qe_run` — different binary, same
flow). The bottom log pane streams stdout: Yahoo cache fills the
first time (~5–10 s), subsequent saves take <1 s.

> **What you should see:** the log pane prints
> `[qe_factor] loading 9 symbols…`, then per-factor IC summary
> lines, then `wrote out/first_factor_report.json`.

## 4. Switch to F7 FCTR

The F7 panel mtime-watches the report file — by the time you
press F7, the IC table is already populated. If it isn't, open
**Settings (Cmd+,) → Research → Factor report path** and point
it at `out/first_factor_report.json` (one-time setup; the
dashboard remembers it).

> **What you should see:**
> - **Top strip**: schema v3, mtime "just now", universe = 9,
>   horizons = [1, 5, 20].
> - **IC table**: three rows. The `t-stat` and `verdict` columns
>   are colored — green = `signal±`, amber = `weak±`,
>   dim = `noise±`. The `±` suffix tells you the trade
>   direction (a `signal−` means *short* the high-momentum names).
> - **Long-short metrics strip**: total return, Sharpe, max DD,
>   turnover for the quintile spread.
> - **Equity curve**: shaded green above 1.0, red below.
> - **Per-bar XS IC plot**: a noisy line bouncing around zero.
>   Hover any point for `date · IC` at that bar.

If every row reads `noise`, your factor isn't predictive on this
universe — that's a useful negative answer, not a bug. Try
`rsi_14_inv` instead (`expr = rsi(close, 14) * -1.0`) for a
clearer signal.

## 5. Opt into walk-forward IC

Add one line inside `research(...)`:

```qe
walk_forward = walk_forward_ic(window_bars = 252, step_bars = 21),
```

Place it just before `output = ...`. `Cmd+S` again. qe_factor
runs ~108 windowed IC slices; you'll see a few more seconds in
the log, then the panel updates.

## 6. Read the walk-forward panel

Below the per-bar XS IC plot, a new panel appears: **walk-forward
xs IC**. Same horizon tab strip at the top.

> **What you should see:**
> - **Dim horizontal at 0** — no-signal baseline.
> - **Muted horizontal at the full-sample IC** — your headline
>   number, for comparison.
> - **Rolling line** — colored by the full-sample t-stat bucket.
> - **Per-window dots** — each colored by *that window's own*
>   t-stat bucket. Clusters of green dots = the signal was
>   actually working in that period; dim clusters = the line
>   was meaningless even when the headline says "signal".
> - **Trend-slope badge** in the header — green / red /
>   "~0 (no consistent trend)" depending on OLS slope + R².
> - Hover any point for the window's date range, IC mean,
>   t-stat, and observation count.

## 7. Read the verdict

Three patterns to look for:

| You see | The factor is… |
|---|---|
| Dots mostly green across all years, slope ~0 | **Stable** — the headline IC is real and persistent. |
| Green dots cluster early then turn dim/amber, slope red | **Decaying** — used to work; market has adapted. |
| Dots flip green ↔ dim in alternating regimes | **Regime-dependent** — useful only with a regime filter. |

A factor with **60+/108** windows above `|t-stat| > 2` (counted
from the JSON, see
[`factor-research.md` § Reading the JSON yourself](factor-research.md#reading-the-json-yourself))
is much stronger evidence than a single full-sample `t-stat = 4`.

## What's next

- Add a second factor — `factor("rsi_14_inv", expr = ...)` — and
  compare them head-to-head in the F7 dropdown.
- When you find one that clears the bar, promote it to a
  `signal(...)` and validate end-to-end (with costs) via
  `qe_run`. The reference is
  [`factor-research.md` § Workflow](factor-research.md#workflow).
- Full grammar for `.qe`: [`qe-language.md`](qe-language.md).
