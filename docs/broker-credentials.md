# Broker credentials

How to store your Alpaca API keys in the OS keyring so the
dashboard, daemon, and CLI tools can read them without leaving
plaintext on disk.

## Quick start

```bash
# Store paper-trading credentials with a Touch ID gate (macOS).
qe_creds set alpaca-paper key_id     YOUR_KEY_ID     --require-auth
qe_creds set alpaca-paper secret_key YOUR_SECRET_KEY --require-auth

# Verify the keychain entry exists (does NOT print the value).
qe_creds list

# Launch the dashboard. The first broker action of the session
# fires one Touch ID prompt; subsequent reads inside the next 5
# minutes are silent.
qe_dashboard
```

For **live** (real-money) trading, replace `alpaca-paper` with
`alpaca-live` and set `ALPACA_LIVE_TRADING=I_KNOW_WHAT_I_AM_DOING`
in your shell before launching the dashboard.

## Two ways to manage credentials

### Command line — `qe_creds`

| Command | What it does |
|---|---|
| `qe_creds set <service> <account> <value> [--require-auth]` | Store or overwrite a credential. `--require-auth` adds a Touch ID gate on every read. |
| `qe_creds get <service> <account>` | Print a credential to stdout (fires Touch ID if gated). |
| `qe_creds list` | Print `service \t account` rows. Never prints values. |
| `qe_creds remove <service> <account>` | Delete a credential. |

Add `--quiet` to silence the progress log lines.

### Dashboard GUI — Settings → Security → Manage credentials

Open Settings with `Cmd+,`, scroll to **Security**, click
**Manage credentials...** The modal mirrors the CLI:

- Lists every stored `(service, account)` pair.
  **Values are never displayed** — to rotate a key, remove the
  entry and re-add it.
- "Add or overwrite a credential" form with a preset dropdown
  covering the four Alpaca paper/live + key/secret combos.
- Value field is masked by default; a **Show** toggle reveals it
  for typo verification.
- **Require Touch ID on read** defaults on.
- **Remove** is a two-click confirmation — the button turns red
  and reads **Confirm?** before deleting.
- Backend badge shows which keyring the OS gave us
  (`macos-keychain` on macOS, `libsecret` on Linux, `memory` if
  neither is available — that last one is a warning state).

Entries created from the CLI or the GUI are interchangeable; both
write through the same keyring.

## Naming conventions

Use these `(service, account)` pairs so the dashboard's broker
resolver finds them:

| service | account | What it's used for |
|---|---|---|
| `alpaca-paper` | `key_id` | Alpaca paper API Key ID |
| `alpaca-paper` | `secret_key` | Alpaca paper Secret Key |
| `alpaca-live` | `key_id` | Alpaca live API Key ID |
| `alpaca-live` | `secret_key` | Alpaca live Secret Key |

You can store other services under any name you like, but only
`alpaca-paper` / `alpaca-live` are wired into the dashboard's
broker connector today.

## Rotating a key

Run `qe_creds set` over the existing entry. The old value is
overwritten in place; the Touch ID marker is re-applied if you
pass `--require-auth`.

```bash
qe_creds set alpaca-paper key_id NEW_KEY_ID --require-auth
```

From the GUI: remove the existing entry, then add it again with
the new value.

## How the dashboard resolves credentials

Every code path that needs Alpaca credentials goes through the
same resolver:

1. **Keyring lookup** — read `alpaca-paper` (or `alpaca-live`)
   with accounts `key_id` and `secret_key`. On macOS, if the
   entries were saved with `--require-auth`, this fires a Touch
   ID prompt the first time per 5-minute window. On success the
   broker connects and a log line reads
   `broker: alpaca creds resolved via SecretStore`. On cancel,
   the dashboard surfaces a banner and the broker stays
   disconnected; clicking **Connect** again retries.
2. **Shell env vars** — falls back to `ALPACA_KEY_ID` /
   `ALPACA_SECRET_KEY` from the process environment. Useful for
   CI and one-shot runs; not recommended for everyday use.
3. **Neither** — no broker. The dashboard stays in offline-only
   mode (backtests, factor research, sweeps still work).

## Recovery: dismissed prompt or stuck on Touch ID

Retry **Connect** after a cancelled prompt. For launch lock recovery and
the limits of `QE_NO_LAUNCH_LOCK`, see
[Credentials security — Recovery](credentials-security.md#recovery).

## IBKR credentials — nothing to store today

Log into TWS or IB Gateway in IBKR's UI. QE connects to that authenticated
process over TCP; it does not store an IBKR password or API key.
Host, port and client ID are connection settings. See
[IBKR connectivity](ibkr-connectivity.md) for setup and
[`ibkr_connection(...)`](qe-language.md#ibkr_connectionhost-port-client_id-account)
for daemon configuration.

## See also

- [credentials-security.md](credentials-security.md) — the
  Touch ID gate, the macOS keychain ACL prompt, the launch-time
  gate option, recovery flows, threat model.
- [code-signing.md](code-signing.md) — signing posture (optional
  for the gate, required for redistribution).
