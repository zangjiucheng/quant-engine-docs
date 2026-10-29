# QE Fleet walkthrough

End-to-end tour of **QE Fleet.app**: connect → watch → research →
edit → deploy.

Two sentences about what the app is for, because they explain most of
the design decisions below:

- **Strategies are authored here and run there.** Your Mac is the
  control plane; the always-on host is the execution plane. Fleet runs
  an agent on each and routes every operation to the machine that can
  honestly answer it.
- **It places no orders.** The daemon submits what its strategy
  decides. The one write Fleet can send to a venue is a cancel, and
  only for a book whose manifest on the host names a control socket.
  There is no order ticket, and none is planned.

---

## 0. Install

See [Download](download.md) for the published `.dmg`, or build the
working tree:

```bash
scripts/install-fleet.sh
```

Fleet shells out to `qe_run --check` and `qe_factor --check` when you
edit a `.qe` file, so install the engine's command-line tools too.
They are no longer inside an app bundle:

```bash
scripts/install-engine-cli.sh --build
```

That puts `qe_run`, `qe_factor` and `qe_creds` in
`~/.local/libexec/qe-engine-<version>-g<sha>/` with symlinks in
`~/.local/bin/`. Fleet finds them through its `qe_bin_dir` setting,
which defaults to `~/.local/bin`.

## 1. The two locks

Both are off by default and both are set on **F9 CONN**. They answer
different questions, and the first one alone leaves the larger gap.

**Touch ID at launch** (`require_launch_auth`). Fleet asks before the
window appears. A refusal is logged and the app exits; it does not
open read-only.

**The window lock.** `⌘L` covers the window immediately, and an idle
interval — 5 minutes, 15, an hour — covers it after that long with no
pointer, key, scroll or touch. This is the one that matters once the
app is open: launch auth guards a single moment, and a window left
open all afternoon is the case it misses.

The cover is opaque and sits over everything, including the context
strip that carries the day's P&L. But a cover is not a lock, and the
distinction is deliberate: while locked, the app **refuses every
operation that writes** — the four venue writes, and saving or
running a `.qe` config — in the shell, not in the page. A key the
palette still has bound, a retry scheduled before the lock or a
callback that outlives the overlay all hit the same refusal. The
screens keep reading, so they are current the moment the cover comes
off.

Unlocking asks for Touch ID and never reuses an earlier one. If this
Mac cannot offer the prompt at all, the window **stays locked** — the
opposite of what the launch gate does, and on purpose: at launch the
alternative is an app that will not start, here it is an app that
stays covered, and the data is already behind the cover. Quit it.

If you are locked out of the **launch** gate — a broken sensor, a
borrowed machine — start once with the bypass:

```bash
QE_NO_LAUNCH_LOCK=1 "/Applications/QE Fleet.app/Contents/MacOS/qe-fleet-app"
```

Then turn the setting off on **F9 CONN** if you do not want it. It is
deliberately the *launch* bypass only: it will not unlock a covered
window, because one environment variable disabling the lock for a
whole session is a second way to fail open.

## 2. F9 CONN — point it at the host first

Nothing else works until this screen does, so it is the first stop
rather than the last.

CONN groups its fields **by machine and names the machine in each
heading**. That is not decoration. A workspace listing taken on the
host looks exactly like one taken on your Mac, and the difference
matters enormously — so the screen never shows you a path without
telling you whose filesystem it is on.

| Lives on the host | Lives on this Mac |
|---|---|
| the fleet manifest | the `.qe` workspace |
| the collector's observation files | `qe_run` / `qe_factor` (`qe_bin_dir`) |
| the daemon's read-only control socket | the index universe |
| reports from studies that ran there | market-data fetches |

Set the SSH host (`nixos` — one name, which auto-selects LAN or
Tailscale) and the fleet root. Fleet starts the remote agent over SSH
at `~/.local/bin/qe-fleet-agent`; deploy it with
`scripts/deploy-fleet-agent.sh` if it is missing.

Asking the wrong agent is an **error**, not a fallback. The refusal is
server-side, by operation name, in the agent's own role table.

## 3. F1 FLEET — what is running

Two tabs over one object.

**OVERVIEW** collapses by comparison group rather than listing every
book: the live fleet has one broker instance and 120 shadow books
across 40 groups, and 121 cards is not a monitoring screen. Click a
group to expand it to its arms. Badges say where attention is needed.

**COMPARISON** answers which results may honestly be put side by side,
and mostly which may not. It adds no metric — no CAGR, no Sharpe, no
ranking. Those would have to be derived from a five-minute snapshot
series, which cannot carry them, and a league table is exactly what
makes an experiment arm look like a funded strategy.

An instance's kind is read from the manifest's `source` field, so a
book is labelled `BROKER PAPER`, `LIVE`, `SHADOW`, `NEWS PAPER` or
`UNKNOWN` — never guessed. `UNKNOWN` means the manifest did not say,
which is a thing to go and fix, not a thing to render as paper.

## 4. F2 INSTANCE — one book in detail

Positions, working orders, evaluation rounds, the equity series and
the session's move.

No quotes on this screen, so the day column has no previous close to
compare against and renders a dash. That is the truth; **F3 MARKET**
is where quotes live. A dash is not a zero, and the two are drawn
differently everywhere in this app on purpose.

The strip above the panels carries the session's move and the serving
record: how many distinct sessions the book has reported, the first
and last, and any gap in between. Gaps are counted in weekdays, so a
weekend is not a gap and a three-week hole is not hidden by one.

## 5. F3 MARKET and F8 MAP — quotes

MARKET is the watchlist: live Yahoo quotes, grouped by GICS sector
from the index universe, with symbols the universe does not carry
collected under "not in the index" rather than dropped into a default
bucket. Drag to reorder; drag across a sector band to pin a symbol
there.

MAP is the treemap of the same universe, sized by weight and coloured
by move. Its height is measured from the window rather than assumed,
so it never grows a scrollbar.

Both read public HTTP directly from this Mac. They do not detour
through the host, because there is no reason for them to.

## 6. F4 BCKT and F7 FCTR — reading what the engine wrote

**BCKT** renders a `results.json` from `qe_run`: metrics, the equity
curve, the trade list, and the per-strategy attribution for a
portfolio run. Historical results carry their own context and never
borrow a live account's values.

**FCTR** (RESEARCH) renders a `factor_report.json` from `qe_factor` —
why a candidate was chosen or refused. What arrives is a
*projection*: a real report is up to 15 MB of per-bar series, and the
reader drops those and downsamples the equity curve so the screen
reads tens of kilobytes. It is not the file and does not pretend to
be; the path is on screen for anyone who wants the file itself.

A `research(...)` study — a parameter grid — is drawn as a grid, with
axis names taken from the study's own declarations and verified
against the cells before use. Sort by any axis; each column carries
its own precision.

## 7. F5 RISK — what is capped and what can be lost

Four tabs, split by the question rather than by how many panels it
takes to answer one:

| Tab | Question |
|---|---|
| EXPOSURE | what is held, and where it is concentrated |
| LIMITS | what is rejecting orders, and what it measures |
| VAR | what the book is worth and how much of it can be lost |
| FORECAST | what drives that |

The limits tab is worth one warning. A strategy-sized cap must never
be measured against account totals: the account holds positions you
manage by hand, the strategy holds its own book, and "raise the cap"
is not a fix for a scope error. The screen names which of the two
each number measures.

## 8. F6 TRADE — what is working, and the one thing you can stop

Working orders, fills, and the blotter.

The only write is **cancel**, and it needs two gates to be reachable:
the book's manifest on the host must name a `control_socket`, and the
agent's own vocabulary must serve the operation. Until both are true
the control is dormant and says so, rather than failing when pressed.

A kill switch trips the daemon's own switch over that socket. It is
one-way by contract: once tripped it stays tripped for the daemon's
process lifetime, `unkill` refuses, and clearing it is an SSH session
and a restart. The restart forces a cold warmup that can itself
refuse to start, so this is not a toggle and the screen does not
offer it as one. A trip blocks **new submissions and cancels
nothing** — working orders stay at the venue, and the panel will
never claim the book was flattened.

## 9. F10 EDIT — the only place Fleet writes

The workspace editor: a file tree, a `.qe` editor with syntax
highlighting, and `--check` on save.

`qe_run --check` is a parse and type-check only. It will not tell you
whether a `live(...)` config is deployable — it accepted configs that
the daemon then rejected for an unknown keyword argument. Check is a
fast no, not a yes.

Writes are confined to the configured workspace root. A path that
escapes it, resolves through a symlink out of it, or exceeds the size
cap is refused by the agent, by path, before anything is written.

Deploying sends the checked config to the host. There is no "deploy
from here and hope": the destination is the host's fleet root and the
operation is served only by the host's agent.

## 10. F11 ANLZ — the news and LLM analysis pipeline

A separate service on the host reads news feeds, filings and quotes,
asks a model to read what it collects, and records what the model said
next to what the policy did with it. ANLZ is the window onto that
service. It replaced a web page the service published on the LAN.

Everything on the screen comes from one snapshot the service writes on
a one-minute timer, and the screen adds no judgement of its own: no
sentiment score, no aggregate confidence, no "the pipeline is
healthy". What it adds is the thing the old page structurally could
not show — **the difference between a quiet pipeline and a stopped
one.** That page was rendered *by* the publisher, so it was fresh by
construction; a reader could not tell a slow news day from a dead
collector.

So the first two things ANLZ says are about the snapshot and not about
the news:

- **the pipeline's own age**, from the publisher's `generated` stamp.
  Past fifteen minutes it says so, and names the publisher rather than
  the news. A snapshot carrying no stamp reports that its age is
  *unmeasured*, which is neither fresh nor stale.
- **the five worker states** — `collect`, `verify`, `analyze`,
  `quotes`, `paper` — as `systemctl --user is-active` reported them. A
  role the snapshot never mentioned reads `absent`, not `active`: a
  worker dropped from the publisher's list would otherwise just stop
  having a badge, and four green badges read as fine.

That strip sits above the tabs and does not scroll, because it
qualifies all of them: every number below is only as good as the
snapshot's age and those five states.

The rest is four tabs, not one column. The content is not one list —
it is the pipeline's **input**, what it **read**, what **came out**,
and how the **machine** is running, and those are four questions
asked at different times:

| Tab | What it answers |
|---|---|
| **News** | what it read, and how the model read it |
| **Signals** | what came out — symbol signals, and the paper ledger |
| **Sources** | what goes in — the 55-odd feeds, worst first, and the latest quotes |
| **Pipeline** | the machine — counters, the model-call budget, why events were *not* analyzed, publication-time verification |

News is a list on the left and the full reading on the right: the
model's summary, the facts it extracted, the passages it quoted, the
uncertainty it stated, and any instrument targets. **Open in
browser** hands the article to your default browser — the window
itself refuses anything that is not an http(s) link, by scheme,
before the browser is reached.

Two things the screen is careful to say plainly. A verified
publication time means the publisher *states* it, not that the story
is true. And `paper` is an observation ledger — simulated at a fixed
notional with commission and slippage, read by nothing, an input to
nothing that trades.

It is read-only in the strongest sense available: nothing here places
an order, and the browser or the window being closed changes nothing
about what the service does.

## 11. Keyboard

Use these workspace shortcuts:

```
F1 FLEET   F2 INSTANCE  F3 MARKET  F4 BCKT  F5 RISK
F6 TRADE   F7 FCTR      F8 MAP     F9 CONN  F10 EDIT
F11 ANLZ
```

`⌘`-digit shortcuts cover F1 through F9.

| Key | Action |
|---|---|
| `j` / `↓` | next row |
| `k` / `↑` | previous row |
| `Enter` | open the selected row |
| `⌘K` or `:` | command palette |
| `?` | help for this screen |
| `Esc` | close, or leave a field |

With vim keys enabled (F9 CONN), `g g` / `G` jump to the first and
last row and `/` then `n` / `N` search the current list.

## 12. When something looks wrong

A few readings that are not bugs, in rough order of how often they
come up:

- **"Not yet compared" after an off-hours restart.** Reconciliation
  is market-hours-only; that is the designed UNKNOWN. Check
  `reconcile_passes ≈ uptime/60`.
- **A paper book's prices look ~15 minutes stale.** They are.
  `ibkr-paper` is IBKR market-data type 3, so the 16:00 evaluation
  ranks on roughly 15:44 prices.
- **A refused daemon restart.** The historical-data farm gates it.
  One historical request — the probe — is the fix, not restarting IB
  Gateway. A TWS 2107 notice does not mean the farm is dead.
- **The daemon shows as gone while it keeps trading.** The
  observation channel and the trading channel are deliberately
  independent. A dead SSH forward takes out the first and not the
  second.
- **Every number reads zero.** Check whether it is an *absent* zero
  or a *measured* one. Fleet draws the three cases differently — no
  source, source present but silent, and a real zero — precisely
  because a default that returns `0` is indistinguishable from a
  measurement that found nothing.

---

## See also

- [Download](download.md) — the published build and the quarantine command
- [Install & setup](install.md) — the engine CLIs and broker credentials
- [Daemon runbook](live-trading-runbook.md) — what runs on the host
- [Safety model](live-trading-safety.md) — what stands between a config and an order
- [Dashboard API (read-only)](dashboard-api.md) — the same contract, over HTTP
