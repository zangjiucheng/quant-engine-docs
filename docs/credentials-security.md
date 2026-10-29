# Credentials security

How this project stores broker API keys, when it asks the OS for
authentication, and how to recover if you lock yourself out.

Two programs read these credentials: **`qe_creds`** on your Mac and
**`qe_daemon`** on the host. A third, **QE Fleet**, has a launch-time
Touch ID gate of its own and never reads a broker credential at all.

!!! note "Persistent storage identifiers"

    The keychain service prefix `qe_dashboard:` and the credentials fallback
    directory `~/Library/Application Support/qe_dashboard/` are current
    storage identifiers. Keep them unchanged: renaming them without migration
    would make saved credentials inaccessible. They are not executable names.

## Where credentials live

Broker credentials live in the **OS keyring**, never in
`results.json`, an app config, or a `.env` file. Shell-exported
`ALPACA_KEY_ID` / `ALPACA_SECRET_KEY` still work as a fallback for CI
and one-shot runs, but the recommended path is `qe_creds set`, or
`scripts/setup-broker.sh` which calls it for you.

| Platform | Backend | Location |
|---|---|---|
| macOS | Security framework (`SecItem*`) | Login keychain. Service prefix `qe_dashboard:`. Visible under `Keychain Access.app` → Login. |
| Linux | libsecret over D-Bus | Default collection (usually GNOME Keyring or KWallet). Schema `io.quant-engine.dashboard`. List with `secret-tool search service qe_dashboard:`. |
| Tests / headless | In-memory map | Process lifetime only, never persisted. |

Strategy files (`.qe`), `results.json`, app config and positions
snapshots are **not** encrypted. They stay readable in the workspace
folder so you can grep them, back them up with ordinary tools, and
inspect them with nothing running.

## The two macOS prompts

On macOS you may see **two different prompts** when `qe_creds` reads a
credential. They look similar but come from different places — knowing
which is which makes recovery easier.

### Prompt 1 — Touch ID

```
┌──────────────────────────────────┐
│  Touch ID                        │
│                                  │
│  quant-engine wants to read your │
│  saved broker credentials.       │
│                                  │
│  [Use Password]       [Cancel]   │
└──────────────────────────────────┘
```

This fires when:

- The credential was saved with `--require-auth`
- The reading process is reading it for the first time in the current
  5-minute window

Touch the sensor (or type your Mac account password as fallback).
Subsequent reads inside the same process are silent for the next 5
minutes. A successful *prompt* restamps the window, but a silent read
inside it does **not** — so the 5 minutes run from the last time you
actually touched the sensor, not from the last read. A long-running
process re-prompts every 5 minutes; it does not stay unprompted
indefinitely. A new process always re-prompts.

### Prompt 2 — "… wants to use your keychain"

```
┌──────────────────────────────────────┐
│  qe_creds wants to use the           │
│  "qe_dashboard:alpaca-paper" item    │
│  in your keychain.                   │
│                                      │
│  Password: [_______________]         │
│                                      │
│  [Always Allow]  [Deny]  [Allow]     │
└──────────────────────────────────────┘
```

This was macOS's per-app keychain ACL gate, fired by the OS
before our Touch ID code even runs. It used to fire when:

- The credential was saved by one binary and is being read by
  another (for example, `qe_creds` wrote it and `qe_daemon` is
  reading)
- The reading binary was rebuilt and the new signature didn't match
  the trusted-apps list captured at write time — every reinstall used
  to require a full credential re-save

Current behaviour: credentials are stored with a **permissive
decrypt ACL** (no per-binary trust list). Reads never trigger the
keychain prompt regardless of which binary is calling. The Touch
ID gate at the application layer (see below) is the actual
authorization check — mark items `--require-auth` to keep it in
the loop. The login keychain itself still needs to be unlocked
(usually auto-unlocked at login) and a foreign process still
can't silently rewrite the ACL.

If you saved credentials before this fix landed, the items still
carry the old restrictive ACL — **re-save them once** and the
prompt goes away for good:

```bash
# From the CLI
KEY=$(qe_creds get alpaca-paper key_id)
SEC=$(qe_creds get alpaca-paper secret_key)
qe_creds set alpaca-paper key_id     "$KEY" --require-auth
qe_creds set alpaca-paper secret_key "$SEC" --require-auth
```

There is no GUI for this. QE Fleet does not read, write or display
broker credentials — the keys stay in the terminal, which is where
`scripts/setup-broker.sh` puts them and the only place they appear.

## When the Touch ID gate fires

The gate fires **lazily** — on the first broker credential read, not
at startup. Offline workflows (backtest, sweep, factor research,
reading results) never see a Touch ID prompt; the dialog appears when
`qe_creds get` runs, or when the daemon opens a broker session.

| Action | Result |
|---|---|
| Touch the sensor (or type Mac password) | The read succeeds. Touch ID is cached for 5 minutes within that process. |
| Cancel | The read fails with a refusal the caller reports. Nothing else is affected. |
| Type the wrong password three times | OS falls back to password retry; eventually treated the same as Cancel. |

## QE Fleet's two gates

That lazy model is about *credentials*. QE Fleet holds none, so its
gate is a different thing: a banking-app-style "confirm you're you
before the window opens" check. Opt in via:

- **F9 CONN → Touch ID at launch**, or `require_launch_auth: true`
  in `~/Library/Application Support/qe-fleet/config.json`.
- The prompt is raised before the window is created, so a cancel
  leaves no flash of the thing the lock exists to hide.
- Cancel or repeated failure exits with status 0 and one line on
  stderr. Declining to unlock your own app is not an error.
- `QE_NO_LAUNCH_LOCK=1` bypasses it, and the bypass is logged.
- **Every outcome is logged, success included.** A gate that is
  silent when it works cannot be told apart from one that never
  ran: `qe-fleet: launch authentication authenticated after 2.0s`.

### The window lock

The launch gate guards one moment. `⌘L`, and an idle interval set on
F9 CONN, guard every moment after it — which is where a window left
open all afternoon actually sits.

- The cover is opaque and over everything, including the strip that
  carries the day's P&L.
- **The cover is not the lock.** While locked, `Fleet::ask` refuses
  every guarded op in the shell: the four venue writes plus
  `write_config` and `run_config`. Which ops those are is read from
  the op vocabulary table, not a second list. Reads are not refused,
  so the screens are current when the cover comes off.
- Unlocking uses a prompt that **never reuses an earlier success**,
  and locking voids any cached one. `qe_mac_auth::prompt` reuses for
  five minutes, which is right for several credential reads in a row
  and wrong here: a lock asks whether the *person changed*, and the
  likeliest other person is whoever sat down just after you left —
  the same five minutes. The first build shipped with that defect and
  opened to a click.
- `QE_NO_LAUNCH_LOCK=1` does **not** unlock a covered window. It is
  the launch bypass; honouring it here would let one environment
  variable silently disable the lock for a session.
- A system that cannot offer the prompt leaves the window **locked**,
  unlike the launch gate below. At launch the alternative is an app
  that will not start; here it is an app that stays covered, and the
  data is already behind the cover.

Two things worth knowing about how the launch gate fails:

- **A system that will not offer the prompt is not a refusal.** No
  Secure Enclave, no enrolled finger, a Mac with the sensor
  disabled — the app continues rather than locking you out of your
  own machine. It treats *no answer at all* the same way: a gate
  that runs before any window exists cannot rely on a human noticing
  a dialog, so it gives up after two minutes and opens with a
  warning rather than hanging forever.
- **It guards the window, not the data.** The fleet observation
  files and the daemon's read-only control socket are protected by
  the filesystem and by SSH either way. What this stops is someone
  walking up to an unlocked Mac, reading the book and pressing the
  one button that cancels orders.

**When to enable**: shared machines, or when you don't want position
sizes glanceable.

**When NOT to enable**: scripted launches.

## Bypass env var: `QE_NO_LAUNCH_LOCK=1`

Setting `QE_NO_LAUNCH_LOCK=1` in the environment:

- Causes new `qe_creds set` calls to skip the Touch ID marker
  (write-side bypass — the new entry has no gate).
- Does **not** affect already-gated items on disk. Those still
  prompt because the marker is a keychain attribute, not a
  process state.
- Skips QE Fleet's launch-time gate if it was on, and logs that it
  did. A bypass nobody can see in a log is a bypass nobody will
  notice was left on.

Use this for:

- **CI runs** — no TTY for the prompt, the dialog would hang.
- **First install** — save the initial credential set without a
  gate, so nothing prompts before you have decided whether you want
  one.
- **Recovery** — see below.

## Recovery

### "I cancelled and now the broker read fails"

1. Run the command again; the prompt comes back.
2. To permanently drop the gate from one entry, re-save it under
   the bypass:
   ```bash
   QE_NO_LAUNCH_LOCK=1 qe_creds set alpaca-paper key_id <YOUR_VALUE>
   ```

### "I locked myself out of QE Fleet at launch"

1. Launch once with the bypass. The env var strips the launch prompt
   without touching the on-disk config:
   ```bash
   QE_NO_LAUNCH_LOCK=1 "/Applications/QE Fleet.app/Contents/MacOS/qe-fleet-app"
   ```
2. **F9 CONN** → untick **Touch ID at launch**, then restart
   normally.

### "Touch ID itself is broken and I forgot my password"

If both biometrics and the password fallback are gone, the
keychain item is unrecoverable.

- **macOS**: open Keychain Access.app, search for
  `qe_dashboard:`, delete the items, then re-pair with the
  broker.
- **Linux**: `secret-tool clear service qe_dashboard:alpaca-paper
  account default` (substitute the service / account that's
  stuck). Then re-save.

## What the gate does NOT protect

- **Strategy files** (`.qe`) — readable by anyone with disk
  access.
- **Backtest results** (`results.json`) — same. The metrics leak
  some signal but not the strategy itself.
- **App config** (`~/Library/Application Support/qe-fleet/config.json`) — plain JSON.
- **Positions snapshots** (`positions/*.json`) — plain JSON.
- **The reading process's memory** — once a credential is in RAM, an
  attacker with debugger access can read it. The gate protects
  against "someone uses my unlocked Mac while I'm at lunch", not
  against rootkits.

If those plaintext files matter for your threat model, layer
file-system encryption underneath: FileVault on macOS, LUKS or
per-directory encryption on Linux.

## iCloud Keychain sync

macOS items are saved with the "accessible when unlocked" policy,
so credentials sync across your Macs via iCloud Keychain when
that's enabled. The Touch ID marker syncs with them, so the
prompt fires on whichever Mac reads the synced credential.

If you want device-local credentials only (no iCloud sync), the
simplest path today is to disable iCloud Keychain at the OS
level for the `qe_dashboard:` items via Keychain Access. A
per-item "device only" flag will arrive in a future release.

## See also

- [broker-credentials.md](broker-credentials.md) — the practical
  `qe_creds` workflow, naming conventions, rotation.
- [code-signing.md](code-signing.md) — signing posture (optional
  for the gate, required for redistribution).
