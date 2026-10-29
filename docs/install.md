# Install & first-time setup

!!! tip "Get the app first"

    [Download QE Fleet for macOS](download.md), then come back here.
    This page covers what happens after the app is on disk: the engine's
    command-line tools, broker credentials, and pointing Fleet at the
    host.

Setup uses the commands below and Fleet’s F9 CONN screen.

## Quick start — first-time user

1. Open the `.dmg` and drag **QE Fleet** to **Applications**.
2. Clear the quarantine flag once — see
   [Download](download.md#macos-will-refuse-to-open-this-the-first-time-here-is-the-fix).
3. Install the engine's command-line tools:

    ```bash
    scripts/install-engine-cli.sh --build
    ```

4. Set up a broker, if you have one:

    ```bash
    scripts/setup-broker.sh
    ```

5. Launch the app and fill in **F9 CONN** — the SSH host and the fleet
   root. See the [walkthrough](fleet-walkthrough.md#2-f9-conn-point-it-at-the-host-first).

Steps 3 and 4 need a source checkout. If you only have the `.dmg`,
Fleet still runs: FLEET, INSTANCE, MARKET, BCKT, RISK, TRADE, FCTR,
MAP and CONN all work without the CLIs. Only **F10 EDIT**'s check and
run need them, because it shells out to `qe_run --check`.

## The engine's command-line tools

`qe_run`, `qe_factor` and `qe_creds` install on their own:

```bash
scripts/install-engine-cli.sh --build   # configure + build the release preset first
scripts/install-engine-cli.sh           # reuse an existing build/release/bin
```

Layout — a versioned directory nothing overwrites, and a symlink that
moves only after the new binaries have run and answered:

```
~/.local/libexec/qe-engine-<version>-g<sha>/qe_run
~/.local/bin/qe_run -> ../libexec/qe-engine-<version>-g<sha>/qe_run
```

The versioned CLI installation is independent of the application bundle.

If `~/.local/bin` is not on your `PATH` yet:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

Check it worked:

```bash
qe_run --version
```

### What it does not install, deliberately

**`qe_daemon`.** It is Linux x86_64 and remote-deploy only. A macOS
`qe_daemon` on your `PATH` is a binary that cannot do the job its name
promises, sitting where something will eventually pick it up. Deploy it
to the host with `scripts/deploy-daemon-nixos.sh`.

The daemon is built and packaged natively on the target host for maximum
performance, including CPU-specific optimization. It has no public binary
or container release. The Linux Fleet Agent is also built and installed
privately on its host (`scripts/deploy-fleet-agent.sh`), with no standalone
Linux download. The macOS local Agent continues to ship inside Fleet.app.

## Broker setup

```bash
scripts/setup-broker.sh                        # interactive
scripts/setup-broker.sh --list                 # what is configured
scripts/setup-broker.sh --check ibkr-paper     # tcp probe + creds check
```

### IBKR Gateway (`ibkr-paper` / `ibkr-live`)

IBKR Gateway authenticates via its own browser-based login flow. There
are no API keys to store. The script checks reachability and prints a
starter `ibkr_connection(...)` snippet for your `live(...)` config:

```qe
ibkr = ibkr_connection(host = "127.0.0.1", port = 4002, client_id = 1)
```

Default Gateway ports: **4002** (paper), **4001** (live). TWS uses
**7497** (paper), **7496** (live). The connectivity probe times out
after 2 s; failure prints the Gateway → API settings checklist (Enable
ActiveX, Socket port, Trusted IPs = 127.0.0.1).

One thing to know before you read a paper book's numbers:
`ibkr-paper` is IBKR market-data type 3, which is **15 minutes
delayed**. A 16:00 evaluation ranks on roughly 15:44 prices.

### Alpaca (`alpaca-paper` / `alpaca-live`)

Generate API keys at
[alpaca.markets/paper/dashboard/overview](https://app.alpaca.markets/paper/dashboard/overview)
(or `/live/` for the live account). The script prompts for `key_id`
and a hidden `secret_key`, then calls:

```bash
qe_creds set alpaca-paper key_id     <YOUR_KEY>     --quiet
qe_creds set alpaca-paper secret_key <YOUR_SECRET>  --quiet
```

Backed by the macOS Keychain / libsecret on Linux. Values are never
printed in any log or UI. The keychain service prefix is
`qe_dashboard:` — a persistent storage identifier; renaming it would
make every credential already stored invisible to the program that
stored it. See [Credentials security](credentials-security.md).

## Verifying the install

```bash
qe_run --version                            # the CLI tools
scripts/setup-broker.sh --list              # configured credentials
scripts/setup-broker.sh --check ibkr-paper  # the broker is reachable
```

To launch:

```bash
open "/Applications/QE Fleet.app"
```

First in-app actions:

- **F9 CONN** — the SSH host and the fleet root. Nothing else reads
  the host until this is set.
- **F1 FLEET** — what is running, collapsed by comparison group.
- **F10 EDIT** — open a `.qe` from the workspace; save runs
  `--check`.
- **F11 ANLZ** — the news and LLM analysis pipeline, if that
  service is deployed on the host. Blank without it, and it says so.

## Reference

- [QE Fleet walkthrough](fleet-walkthrough.md) — every screen, in order.
- [`.qe` language](qe-language.md) — configuration and builtin reference.
- [Daemon smoke checklist](qe-daemon-smoke.md) — deployment verification.
- [Forecasting](forecasting.md) — ridge, lasso and GBDT signals.
- [Broker credentials](broker-credentials.md) — `qe_creds` usage.
