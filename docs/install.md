# Install & first-time setup

Two paths get you from a fresh clone to a deployed daemon:

- **Shell wizard** — `scripts/setup-wizard.sh` orchestrates build /
  install / broker credentials / next-steps in one terminal session.
  Best for the initial install.
- **In-app first-launch modal** — when you first launch
  `QE Dashboard.app`, a setup wizard modal opens automatically and
  walks through workspace / brokers / starter files. Re-runnable from
  Settings (`Cmd+,` → "Re-run setup wizard...").

Both flows are idempotent. Re-running skips steps whose artifacts
already exist (built binaries, installed app, configured creds).

## Quick start — first-time user

```bash
git clone https://github.com/zangjiucheng/quant-engine.git
cd quant-engine
scripts/setup-wizard.sh
```

The wizard:

1. **Build status** — detects whether `build/release/` is fresh; offers
   `scripts/compile.sh release` if stale.
2. **Install location** — `~/Applications/QE Dashboard.app` (macOS,
   no sudo) or `~/.local/` (Linux). Wraps `scripts/install.sh`.
3. **CLI on PATH** — optional symlinks for `qe_daemon` / `qe_run` /
   `qe_factor` / `qe_creds` into `~/.local/bin` so you can invoke
   them from a terminal. The `.app` already finds bundled copies for
   in-app use; this only matters for CLI workflows.
4. **Workspace folder** — defaults to `~/Documents/quant-strategy/`.
   First-launch bootstrap drops starter `.qe` files
   (backtest / sweep / forecast / live) into a fresh folder.
5. **Broker credentials** — calls `scripts/setup-broker.sh` for an
   interactive multi-select. Per broker:
   - `ibkr-paper` / `ibkr-live`: prompts for host/port; runs a TCP
     connectivity probe against IBKR Gateway. No API keys to store.
   - `alpaca-paper` / `alpaca-live`: prompts for `key_id` + `secret_key`;
     stores via `qe_creds set <service> <field> <value>` (OS keyring).
6. **Next steps** — prints the launch command and a one-line tour of
   F3 / F6 / Cmd+,.

## In-app first-launch modal

Launching the dashboard for the first time (or with a config where
`setup_completed == false`) opens a 5-step modal:

1. Welcome
2. Workspace path
3. Broker selection (checkboxes: ibkr-paper / ibkr-live /
   alpaca-paper / alpaca-live)
4. Credentials — for each checked broker, opens a Terminal window with
   `scripts/setup-broker.sh --broker <name>` so API keys stay in the
   terminal (never in the dashboard's text boxes).
5. Done

Skip the wizard entirely with **"Skip & don't show again"**. Re-run
later via `Cmd+,` → "Re-run setup wizard".

## Manual install (no wizard)

If you prefer the existing scripted flow:

```bash
scripts/compile.sh release         # build
scripts/install.sh --with-cli      # macOS: .app to ~/Applications + CLI to ~/.local/bin
scripts/install-cli.sh             # interactive PATH setup (subset of setup-wizard.sh)
scripts/setup-broker.sh            # broker credentials only
```

`scripts/install.sh` flags:

| Flag | Effect |
|---|---|
| `-b` | Run `scripts/compile.sh` first |
| `--with-cli` | macOS: also install CLI tools to `~/.local/bin` |
| `--only app` / `--only cli` | Install just one component |
| `APP_PREFIX=...` env | Override .app destination (e.g. `/Applications`) |
| `CLI_PREFIX=...` env | Override CLI destination |

## Broker setup details

### IBKR Gateway (`ibkr-paper` / `ibkr-live`)

IBKR Gateway authenticates via its own browser-based login flow.
There are no API keys to store. The wizard checks reachability and
prints a starter `ibkr_connection(...)` snippet you can paste into
your `live(...)` config:

```qe
ibkr = ibkr_connection(host = "127.0.0.1", port = 4002, client_id = 1)
```

Default Gateway ports: **4002** (paper), **4001** (live). TWS uses
**7497** (paper), **7496** (live). The wizard's connectivity probe
times out after 2 s; failure prints the Gateway → API settings
checklist (Enable ActiveX, Socket port, Trusted IPs = 127.0.0.1).

### Alpaca (`alpaca-paper` / `alpaca-live`)

Generate API keys at
[alpaca.markets/paper/dashboard/overview](https://app.alpaca.markets/paper/dashboard/overview)
(or `/live/` for the live account). The wizard prompts for `key_id`
and a hidden `secret_key`, then calls:

```bash
qe_creds set alpaca-paper key_id     <YOUR_KEY>     --quiet
qe_creds set alpaca-paper secret_key <YOUR_SECRET>  --quiet
```

Backed by the macOS Keychain / libsecret on Linux. Values are never
printed in any log or UI.

## Verifying the install

After the wizard:

```bash
scripts/setup-broker.sh --list             # shows configured creds
scripts/setup-broker.sh --check ibkr-paper # tcp probe + creds check
qe_daemon --version                        # if you opted into CLI PATH
```

To launch:

```bash
# macOS
open ~/Applications/QE\ Dashboard.app

# Linux
qe_dashboard
```

First in-app actions:
- **F3 WKSP** — open a starter `.qe`, `Cmd+S` to run / register live.
- **F6 TRADE** — Deploy panel when no daemon attached; live blotter
  when a daemon is up.
- **`Cmd+,`** — Settings (re-run wizard, broker credentials,
  refresh interval, vim mode, ...).

## Reference

- `docs/qe-language.md` — `.qe` config language reference.
- `docs/qe-daemon-smoke.md` — daemon deployment + smoke walkthrough.
- `docs/forecasting.md` — `predict_return` ridge / lasso DSL.
- `docs/broker-credentials.md` — full `qe_creds` reference.
