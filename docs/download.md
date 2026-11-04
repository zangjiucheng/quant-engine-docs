# Download QE Fleet

<!-- qe:min-macos 11 -->
<!-- qe:min-macos-cli 13 -->

QE Fleet is the window onto the strategies that are running: which
instances exist, what each one is doing, and which results may
honestly be compared. Nearly everything it does is reading. It can
also stop a strategy, cancel the orders one already has working, send
a config to the host, and push a few of that config's risk caps into
the daemon already running — four operations in all, listed below with
what gates each. **It places no orders and has no button that could.**
Download it, drag it to Applications, run it.

**Apple Silicon Macs only** — M1, M2, M3, M4 and later. There is no
Intel build, no Linux build and no Windows build.

**Requires macOS 11 (Big Sur) or later.**

**It is not notarized.** This is a private build made on the
maintainer's machine, not by a build service, and Apple has not
reviewed it. macOS will refuse to open it the first time; the fix is
one command and it is below. If that is not acceptable to you, do not
install it — that is a reasonable position and the page would rather
say so than bury it.

<!-- BEGIN:release-block -->
[Download QE Fleet 0.5.6 for macOS](https://dl.jiucheng-zang.ca/qe-fleet/0.5.6/QE-Fleet-0.5.6-macos-arm64.dmg){ .md-button .md-button--primary }

| | |
|---|---|
| Version | **0.5.6** |
| Released | 2026-11-04 |
| File | `QE-Fleet-0.5.6-macos-arm64.dmg` |
| Signature | ad-hoc — **not notarized**, see below |

Or from the command line, download and verify in one go:

```bash
curl -LO https://dl.jiucheng-zang.ca/qe-fleet/0.5.6/QE-Fleet-0.5.6-macos-arm64.dmg
curl -LO https://dl.jiucheng-zang.ca/qe-fleet/0.5.6/QE-Fleet-0.5.6-macos-arm64.dmg.sha256
shasum -a 256 -c QE-Fleet-0.5.6-macos-arm64.dmg.sha256
```
<!-- END:release-block -->

`shasum` printing `OK` means the file you have is the file that was
published. It says nothing about who published it — that is what
notarization would say, and this build does not have it.

## Or build it yourself

If you have the repository, you do not need the download at all:

```bash
./scripts/install-fleet.sh
```

From the repository root, on an Apple Silicon Mac with Node/npm,
Rust/Cargo and the Xcode command-line tools. It builds the current
working tree, verifies and signs the bundle, then installs it at
`/Applications/QE Fleet.app`. Quit the app first and reopen it
afterward; the previous one is kept in a backup directory the script
prints. It publishes nothing and does not touch the remote daemon.

To install an already-built bundle without rebuilding, add
`--no-build`. To install somewhere else, add
`--app-prefix "$HOME/Applications"`.

## macOS will refuse to open this the first time. Here is the fix.

Install the app first — open the `.dmg` and drag **QE Fleet** onto the
**Applications** folder — then run this once in Terminal:

```bash
xattr -r -d com.apple.quarantine "/Applications/QE Fleet.app"
```

The `-r` is not optional. Without it the command clears the flag from
the app's outer folder and leaves it on every file inside, which looks
like it worked and does not.

**Why this is necessary.** Everything your browser downloads is tagged
by macOS with `com.apple.quarantine`, and for an app Apple has not
reviewed, that tag means "refuse to run". The command removes the tag
from the copy you just installed. It changes nothing else on your Mac
and affects no other app.

If you skip it, the failure is unhelpful: from Terminal the app is
killed instantly and prints nothing; from Finder you may get a dialog
describing the app as damaged. It is not damaged. That is the message
macOS uses when it cannot verify an app, and deleting the download is
not the fix.

## What is inside

Two programs, and it is worth knowing why:

| | |
| --- | --- |
| `QE Fleet` | The window. It renders and nothing else. |
| `qe-fleet-agent` | Reads the fleet and keeps a warm snapshot. |

They are separate processes on purpose. A fault in the reader costs a
restarted reader rather than a closed window; the cache and any SSH
session outlive a settings change; and the agent can be run **on the
machine that holds the data**, which is the configuration worth having
— a remote command costs about a second before it has read a byte, so
an agent on that host turns every read into a local one. Measured
first-screen time dropped from 4.4 s to 0.23 s.

## What it can change

Three layers sit between the window and a venue, and each one can do
less than the layer below it.

| Layer | What it reads | What it can write |
| --- | --- | --- |
| **QE Fleet** — the window | nothing directly; it renders what an agent hands it | nothing. Every operation is issued by an agent |
| **`qe-fleet-agent`** — one beside the window, one on the host | the fleet manifest, the collector's observation files, your `.qe` workspace, and the daemon's read verbs | saves a `.qe` in the configured workspace root and runs the engine on it, both on your own Mac; sends a `.qe` to the host; sends three verbs that are not reads to a running daemon — `cancel_all`, `kill`, `reload_apply` |
| **`qe_daemon`** — on the Linux host, not part of this download | its broker, its feed, its own journal | places and cancels orders. It is the only thing here that talks to a venue |

So there are four operations this app can start, and **exactly one of
them reaches a venue**:

| Operation | What it does | What it touches | What has to be true first |
| --- | --- | --- | --- |
| **cancel** | cancels every order the daemon finds working | the venue | the book's entry in the host's manifest names a `control_socket` — absent by default — and the agent answering serves the operation |
| **kill** | trips the daemon's own kill switch, so that strategy submits nothing more | one switch inside that process. **Working orders stay working** and the panel will not claim the book was flattened | the same two. One-way: clearing it is an SSH session and a daemon restart |
| **deploy** | writes a checked `.qe` into the directory the running daemon reads | one file on the host | the config is a `live(...)`; the host's own `qe_daemon` checked it there, with the binary that daemon is running; and in the live window the collector's deployment-gate verdict is a `PASS` published within the last fifteen minutes |
| **apply** | pushes a deployed config's *hot* changes into the running gate — position cap, gross cap, daily-loss cap, orders per minute | four numbers in a running process | all of deploy's, plus the same `control_socket`. Every other field is refused by name, and so is an apply while a scheduled slice is still working |

A deploy is a file and an apply is four caps. Neither makes a deployed
strategy the running strategy — that is a restart, which this app does
not do.

Cancel and kill do not render at all on a book whose manifest names no
`control_socket`: not a disabled control, an absent one. Deploy is
offered for any `live(...)` config you have open, so the refusals that
matter for it are the host's — the daemon's own check, and the
deployment gate. In the live window every one of the four also asks for
Touch ID at the moment it is issued, and kill and deploy ask you to
type the book's name first. A locked window refuses all four, plus
saving or running a `.qe` and starting or stopping the live-gateway
session, in the shell rather than in the page.

## What it will not do

**There is no order ticket**, and no button that places an order. Not
a disabled one — an absent one: the vocabulary the window and the
agents share has no verb for it, and there is no generic
`request(op, args)` proxy to carry one in under a name that is
allowed. The daemon holds the same line from its own side: its
read-only socket serves a named list of read verbs and refuses
everything else **by name, per connection**, saying which endpoint
turned the request down rather than implying a permissions system that
does not exist.

It also never claims to know something it did not measure. A number
that could not be read says so, with the reason, instead of appearing
as a zero; a snapshot that is forty seconds old says it is forty
seconds old; and a refresh that failed leaves the previous snapshot in
place rather than emptying the screen.

## First run

Open **Connection** and say where to look: the fleet manifest, the
collector's observation files, and — if you want live daemon detail —
the daemon's read-only control socket.

Every path is relative to **wherever the agent runs**. If you point the
agent at a remote host, it reads that host's filesystem and cannot see
this Mac's; set the research directory to something on that host, or
accept that the research screen has nothing to list.

## Engine command-line tools

Separate from the app, and most people do not need them. QE Fleet
reads what these produce; it does not need them installed to do it.

| | |
|---|---|
| `qe_run` | Run a backtest, sweep or walk-forward from a `.qe` file. Writes `results.json` and a self-contained `report.html`. |
| `qe_factor` | Factor research from a `research(...)` `.qe`. Writes `factor_report.json`. |
| `qe_creds` | Store and check broker credentials in the login keychain. |

**Apple Silicon only, and macOS 13 (Ventura) or later** — a higher
floor than the app's macOS 11. That is not an oversight: Apple's
libc++ marks the floating-point `std::from_chars` as introduced in
macOS 26, and the engine routes every such call through a header that
avoids it. 13.0 is where the rest of the C++20 surface lands. The
publisher reads the floor off this page and refuses a build that
disagrees, so if these two numbers ever drift apart the release stops
rather than shipping a binary that aborts at launch.

**Not notarized either.** Same situation as the app, same fix — see
*"macOS will refuse to open this the first time"* above. Command-line
binaries are quarantined the same way.

<!-- BEGIN:cli-release-block -->
[Download the engine CLIs 0.5.6](https://dl.jiucheng-zang.ca/qe-cli/0.5.6/qe-cli-0.5.6-macos-arm64.tar.gz){ .md-button }

| | |
|---|---|
| Version | **0.5.6** |
| Released | 2026-11-04 |
| File | `qe-cli-0.5.6-macos-arm64.tar.gz` |
| Contains | `qe_run`, `qe_factor`, `qe_creds` |
| Signature | ad-hoc — **not notarized**, see above |

```bash
curl -LO https://dl.jiucheng-zang.ca/qe-cli/0.5.6/qe-cli-0.5.6-macos-arm64.tar.gz
curl -LO https://dl.jiucheng-zang.ca/qe-cli/0.5.6/qe-cli-0.5.6-macos-arm64.tar.gz.sha256
shasum -a 256 -c qe-cli-0.5.6-macos-arm64.tar.gz.sha256
```
<!-- END:cli-release-block -->

Extract and put them somewhere on your `PATH`:

```bash
tar xzf qe-cli-*-macos-arm64.tar.gz
mkdir -p ~/.local/bin && mv qe_run qe_factor qe_creds ~/.local/bin/
qe_run --version
```

If you have the source tree, `scripts/install-engine-cli.sh` does the
same thing from a local build and keeps each version in its own
directory with a symlink, so an upgrade never overwrites a binary that
is running.
