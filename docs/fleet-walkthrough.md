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
  decides. Four operations write anything at all — cancel, kill,
  deploy, apply — and the one write Fleet can send to a venue is the
  cancel, and only for a book whose manifest on the host names a
  `control_socket`. There is no order ticket, and none is planned.

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
operation that writes** — the four writes (cancel, kill, deploy,
apply), saving or running a `.qe` config, and starting or stopping the
live-gateway session — in the shell, not in the page. A key the
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

## Paper and live are two processes

Not two tabs and not a filter. The mode is chosen before anything is
read, it selects a different config file, and the agent is told which
half it serves so that a refusal happens on the host.

```bash
# the paper fleet — also what a plain launch gives you
open -a "QE Fleet"

# the live fleet, as a second process
QE_FLEET_MODE=live "/Applications/QE Fleet.app/Contents/MacOS/qe-fleet-app"
```

| | paper | live |
|---|---|---|
| `QE_FLEET_MODE` | unset or `paper` | `live` |
| config file | `~/Library/Application Support/qe-fleet/config.json` | the same directory's `config-live.json` |
| books it serves | broker-paper, shadow, news-paper | live only |

Four things follow:

- **Paper is untouched.** `config.json` keeps its name and its contents,
  so an operator who never asks for live mode is unaffected by any of
  this. Absent means paper, because saying nothing should land on the
  safe side.
- **An unrecognised value refuses to start.** `QE_FLEET_MODE=LIVE`, or a
  typo, is an error rather than a fall-back to paper: somebody who asked
  for live and silently got paper would be operating a fleet they are
  not looking at.
- **The isolation is server-side.** The agent filters the instance list
  by book kind before any screen sees it, and refuses an op on the other
  half's book *by name* — "this agent is the 'paper' half and 'x' is a
  'live' book". A filter in a window is crossed by a stale callback or a
  bound key; this is not. A book the manifest does not declare belongs to
  neither half and is operable from neither.
- **Live mode proves presence on every write**, with the book named in
  the prompt, and never reuses the unlock's authentication. A Mac that
  cannot offer the prompt refuses the write — the opposite of what the
  launch gate does, and deliberately so: there the alternative is an app
  that will not start.

The live window also carries a permanent `LIVE` band that no screen
scrolls away, and `kill_strategy` / `deploy_strategy` require the book's
id to be typed before they are issued. Two windows side by side is the
whole risk, and a badge that scrolls off is not enough.

**A change to a live book additionally needs a current deployment-gate
verdict.** Deploy and apply are the two ops this stands in front of.
The agent reads the verdict the collector published — through the same
parser the screens use, so the gate a write is refused on is the one
you are looking at — and refuses the change when it is absent,
failing, or older than 15 minutes — three missed collection passes. An unanswered gate is not a passed
one. Cancelling and stopping are deliberately **not** gated: a gate
between an operator and the stop button is a hazard, not a control. See
[the deployment gate](deployment-gate.md) for what the verdict means.

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

## 8. F6 TRADE — what is working, and the two ways to stop it

Working orders, fills, and the blotter.

Two of the four writes live here, **cancel** and **kill**, and both
need the same two gates to be reachable: the book's manifest on the
host must name a `control_socket`, and the agent's own vocabulary must
serve the operation. Until both are true neither control renders, and
the panel says which of the two gates is shut rather than failing when
pressed. The cancel is the one write Fleet can send to a venue.

A kill switch trips the daemon's own switch over that socket. It is
one-way by contract: once tripped it stays tripped for the daemon's
process lifetime, `unkill` refuses, and clearing it is an SSH session
and a restart. The restart forces a cold warmup that can itself
refuse to start, so this is not a toggle and the screen does not
offer it as one. A trip blocks **new submissions and cancels
nothing** — working orders stay at the venue, and the panel will
never claim the book was flattened.

## 9. F10 EDIT — where a strategy is written, and sent

The workspace editor: a file tree, a `.qe` editor with syntax
highlighting, and `--check` on save.

`qe_run --check` is a parse and type-check only. It will not tell you
whether a `live(...)` config is deployable — it accepted configs that
the daemon then rejected for an unknown keyword argument. Check is a
fast no, not a yes.

Saving is confined to the configured workspace root, on this Mac. A
path that escapes it, resolves through a symlink out of it, or exceeds
the size cap is refused by the agent, by path, before anything is
written.

The other two writes are here. **Deploy** sends the checked config to
the host, and there is no "deploy from here and hope": the destination
is the directory the running daemon reads, asked of that daemon, and
the operation is served only by the host's agent. It is offered for
any `live(...)` config you have open, so the refusals that matter for
it are the host's — the daemon's own check, run there with the binary
that daemon is running, and in the live window a deployment-gate
verdict that is a `PASS` published within the last fifteen minutes.

**Apply** then pushes that config's *hot* changes into the running
gate: position cap, gross cap, daily-loss cap, orders per minute, and
nothing else. The daemon refuses every other field by name, and
refuses altogether while a scheduled slice is still working. It needs
the same `control_socket` authorisation the two stops on F6 need. A
deployed strategy does not become the running strategy this way —
that is a restart, which Fleet does not do.

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
- **A book reads neither healthy nor unhealthy.** `healthy` is
  three-valued. A book whose operational state is fine but whose
  valuation is not being refreshed *because the exchange is closed*
  reads unknown, not bad, and carries a note saying so. A genuine
  fault is still red, and the note names which input failed — see the
  [dashboard API](dashboard-api.md#where-the-three-buckets-come-from).
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
- [Deployment gate](deployment-gate.md) — what has to be true before a live book exists
- [Dashboard API (read-only)](dashboard-api.md) — the same contract, over HTTP
