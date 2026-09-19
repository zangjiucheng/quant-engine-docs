# Credentials security

How `qe_dashboard` stores broker API keys, when it asks the OS for
authentication, and how to recover if you lock yourself out.

## Where credentials live

Broker credentials live in the **OS keyring**, never in
`results.json`, your dashboard config, or a `.env` file. Shell-
exported `ALPACA_KEY_ID` / `ALPACA_SECRET_KEY` still work as a
fallback for CI and one-shot runs, but the recommended path is
`qe_creds set` or Settings → Security → "Manage credentials..."

| Platform | Backend | Location |
|---|---|---|
| macOS | Security framework (`SecItem*`) | Login keychain. Service prefix `qe_dashboard:`. Visible under `Keychain Access.app` → Login. |
| Linux | libsecret over D-Bus | Default collection (usually GNOME Keyring or KWallet). Schema `io.quant-engine.dashboard`. List with `secret-tool search service qe_dashboard:`. |
| Tests / headless | In-memory map | Process lifetime only, never persisted. |

Strategy files (`.qe`), `results.json`, dashboard config, and
positions snapshots are **not** encrypted. They stay readable in
the workspace folder so you can grep them, back them up with
ordinary tools, and inspect them without the dashboard running.

## The two macOS prompts

On macOS you may see **two different prompts** when the dashboard
or `qe_creds` reads a credential. They look similar but come from
different places — knowing which is which makes recovery easier.

### Prompt 1 — Touch ID

```
┌──────────────────────────────┐
│  Touch ID                    │
│                              │
│  qe_dashboard wants to read  │
│  your saved credentials.     │
│                              │
│  [Use Password]   [Cancel]   │
└──────────────────────────────┘
```

This fires when:

- The credential was saved with `--require-auth` (the CLI flag)
  or with **Require Touch ID on read** ticked in the GUI form
- The dashboard is reading it for the first time in the current
  5-minute window

Touch the sensor (or type your Mac account password as fallback).
Subsequent reads inside the same dashboard process are silent for
the next 5 minutes. The window resets on every successful prompt,
so an actively-used session stays unprompted indefinitely.
Closing and reopening the dashboard always re-prompts.

### Prompt 2 — "qe_dashboard wants to use your keychain"

```
┌──────────────────────────────────────┐
│  qe_dashboard wants to use the       │
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
  another (for example, `qe_creds` wrote it and the dashboard is
  reading)
- The dashboard binary was rebuilt and the new signature didn't
  match the trusted-apps list captured at write time — every
  reinstall used to require a full credential re-save

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

Or use the GUI: Settings → Security → Manage credentials, remove
each entry, then re-add it via the form.

## When the Touch ID gate fires

The gate fires **lazily** — on the first broker credential read,
not at dashboard launch. Offline workflows (backtest, sweep,
factor research, dashboard browsing) never see a Touch ID prompt;
the dialog appears when you click **Connect to broker** or a live
positions JSON triggers a poll.

| Action | Result |
|---|---|
| Touch the sensor (or type Mac password) | Broker connects. Touch ID is cached for 5 minutes within the dashboard process. |
| Cancel | Dashboard shows the banner "Auth cancelled. Click Connect to retry." Offline screens keep working. |
| Type the wrong password three times | OS falls back to password retry; eventually treated the same as Cancel. |

## Optional launch-time gate

The default gate model is lazy. If you'd rather have a
banking-app-style "confirm you're you at every launch" check, opt
in via:

- **Settings → Security → ☑ Require Touch ID at launch**, or
- Edit `~/Library/Application Support/qe_dashboard/config.json`:
  ```json
  {
    "security": {
      "require_launch_auth": true
    }
  }
  ```

With this on, the dashboard prompts for Touch ID (or password
fallback) **before any window appears**. Cancel exits the
process cleanly (exit code 0, no UI flash). Successful auth
continues into the normal startup path.

**When to enable**: shared machines, or when you don't want
Sharpe ratios and position sizes glanceable on the BCKT screen.

**When NOT to enable**: scripted launches, or if you want
offline workflows uninterrupted.

## Bypass env var: `QE_NO_LAUNCH_LOCK=1`

Setting `QE_NO_LAUNCH_LOCK=1` in the environment:

- Causes new `qe_creds set` calls to skip the Touch ID marker
  (write-side bypass — the new entry has no gate).
- Does **not** affect already-gated items on disk. Those still
  prompt because the marker is a keychain attribute, not a
  process state.
- Skips the launch-time gate above if it was on.
- Triggers a startup warning banner in the dashboard logs so a
  misconfigured deployment is visible at a glance.

Use this for:

- **CI runs** — no TTY for the prompt, the dialog would hang.
- **First install** — save the initial credential set without a
  gate so the dashboard doesn't prompt before you've decided
  whether you want one.
- **Recovery** — see below.

## Recovery

### "I cancelled and now my dashboard can't connect"

1. Click **Connect** in the broker banner again.
2. Or restart the dashboard with `QE_NO_LAUNCH_LOCK=1
   qe_dashboard` to bypass the gate for that session.
3. To permanently drop the gate from one entry, re-save it under
   the bypass:
   ```bash
   QE_NO_LAUNCH_LOCK=1 qe_creds set alpaca-paper key_id <YOUR_VALUE>
   ```

### "I locked myself out of the dashboard at launch"

1. Launch once with `QE_NO_LAUNCH_LOCK=1 qe_dashboard`. The
   env-var bypass strips the launch prompt without touching the
   on-disk config.
2. Settings → Security → uncheck **Require Touch ID at launch**,
   close the modal (auto-saves), restart normally.

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
- **Dashboard config** (`~/Library/Application
  Support/qe_dashboard/config.json`) — plain JSON.
- **Positions snapshots** (`positions/*.json`) — plain JSON.
- **The dashboard process memory** — once a credential is in
  RAM, an attacker with debugger access can read it. The gate is
  protecting against "someone runs my dashboard and clicks
  Connect while I'm at lunch," not against rootkits.

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
  `qe_creds` and GUI workflow, naming conventions, rotation.
- [code-signing.md](code-signing.md) — signing posture (optional
  for the gate, required for redistribution).
