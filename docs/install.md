# Install & first-time setup

!!! tip "Get the app first"

    [Download QE Dashboard for macOS](download.md), then come back
    here. This page covers what happens after the app is on disk:
    workspace, broker credentials, and the first launch.

When you first open `QE Dashboard.app` a setup wizard modal appears and
walks you through workspace folder, brokers and starter files. It is
re-runnable at any time from Settings (`Cmd+,` → "Re-run setup
wizard…"), and it is idempotent — running it again skips anything
already configured.

## Quick start — first-time user

1. Open the `.dmg` and drag **QE Dashboard** to **Applications**.
2. Clear the quarantine flag once — see
   [Download](download.md#macos-will-refuse-to-open-this-the-first-time-here-is-the-fix).
3. Launch the app. The setup wizard opens by itself.
4. Work through workspace → brokers → starter files.

That is the whole install. Everything below is detail on the individual
steps, and on doing them by hand if you would rather not use the wizard.

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

## Using the CLI tools from a terminal

The `.dmg` ships four command-line binaries inside the app bundle, and
the app finds them there by itself — F3's Cmd+S, the F6 Deploy panel and
the factor screens all work with no further setup. You only need this
section if you want to run them from a terminal yourself.

They live in the bundle:

```
/Applications/QE Dashboard.app/Contents/MacOS/qe_run
/Applications/QE Dashboard.app/Contents/MacOS/qe_factor
/Applications/QE Dashboard.app/Contents/MacOS/qe_daemon
/Applications/QE Dashboard.app/Contents/MacOS/qe_creds
```

To put them on your `PATH`, symlink them somewhere already on it:

```bash
mkdir -p ~/.local/bin
for t in qe_run qe_factor qe_daemon qe_creds; do
  ln -sf "/Applications/QE Dashboard.app/Contents/MacOS/$t" ~/.local/bin/$t
done
```

If `~/.local/bin` is not on your `PATH` yet, add it to your shell rc:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

Symlinks rather than copies, deliberately: the next release replaces the
app bundle, and a copy would leave you running an old `qe_daemon`
against a new dashboard without either of them saying so.

Check it worked:

```bash
qe_daemon --version
```

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
open "/Applications/QE Dashboard.app"
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
