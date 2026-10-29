# Running strategy comparisons

`scripts/qe-strategy-fleet.py` prepares independent **Linux / NixOS paper**
instances. A manifest can name any daemon-runnable `.qe`; the launcher does
not know the strategy's factor, universe or rebalance interval. It asks the
native `qe_daemon validate --json` command to bind and validate each config.
It never replaces the default `qe-daemon.service` or adopts its holdings.

## Isolation requirements

Each order-capable instance needs:

- Its own **paper account and dedicated, single-account Gateway endpoint**.
  Different client IDs on one account do not separate net positions. The
  fleet rejects both shared accounts and shared endpoints. Other independent
  trading programs must not manage that dedicated account. A new book must
  start against a verified flat account with no working orders. The launcher
  does not liquidate, adopt or migrate existing holdings for you.
- A stable `strategy_id`, absolute journal under
  `/datapool/qe-data/strategy-fleet/instances/<id>/journal`, and a pinned config.
- A separate XDG runtime directory (socket and process lock), config directory
  (order log), data directory (state and baseline), and state directory (logs).
  Merely changing `journal_dir` is insufficient. macOS ignores these XDG
  paths, so this launcher deliberately refuses macOS deployment.
- Four finite, positive risk limits. Optional declared initial entry budgets
  are checked against the gross and per-name limits. A limit is a protective
  bound, not a sizing rule or a promise that retained holdings are rebalanced.

`ibkr_connection(account = "DU...")` now acts as an **identity guard** on
both daemon connections. The Gateway must announce exactly that one account.
Missing, wrong or multiple accounts refuse startup; a changed account after
reconnect withdraws connectivity and blocks submissions. This is not a
sub-account selector: the transport still cannot filter a multi-account
Gateway's position and order streams. The empty default keeps existing
single-account deployments compatible.

## Manifest and preparation

An instance example (replace paths, hashes and account with verified values):

```json
{
  "schema": 1,
  "storage_root": "/datapool/qe-data/strategy-fleet",
  "binary": "/datapool/qe-data/strategy-fleet/bin/qe_daemon",
  "binary_sha256": "<sha256>",
  "instances": [{
    "id": "it-invvol-half",
    "strategy_id": "it-invvol-half",
    "mode": "paper",
    "account_id": "<independent DU account>",
    "config": "/absolute/path/it-invvol-half.qe",
    "config_sha256": "<sha256>",
    "entry_gross_budget_usd": 12500,
    "entry_position_budget_usd": 1666.6666666666667
  }]
}
```

Register an existing daemon with `mode: "observe"`, its actual `account_id`,
`strategy_id`, config/hash, `socket`, and absolute `journal_path`. It is reserved
against conflicts and observed read-only. It can never be launched by `run`.

```sh
python3 scripts/qe-strategy-fleet.py check --manifest /datapool/path/fleet.json
python3 scripts/qe-strategy-fleet.py prepare --manifest /datapool/path/fleet.json
```

Both commands contact no broker. `prepare` creates an immutable release with
config copies, wrapper/client source, hashes and per-instance service units.
It does **not** install or start them. A changed config needs a new release;
do not overwrite an existing book's journal or reuse an ID for another test.

After account/Gateway setup and review, copy only the desired
`qe-paper-<id>.service` to the user systemd directory, create the parent
`/run/user/$(id -u)/qe-fleet` with mode 0700, reload systemd, and explicitly
enable/start that instance. Each service calls `run`, which repeats validation
and verifies the release before executing the pinned binary. `Restart=no`
keeps an intentional stop stopped. Do not attach the incumbent's supervisor
to a new instance without separately configuring its target and state.

On NixOS, pin the new binary's dynamic `/nix/store` dependencies and the Python
interpreter as GC roots. Do not repoint the incumbent's binary symlink just
to install a comparison instance.

To inspect one instance with the existing Python API:

```python
from qe_daemon import DaemonClient
c = DaemonClient("/run/user/1002/qe-fleet/it-invvol-half/qe_daemon.sock")
print(c.status())
print(c.book())
```

For a remote client, forward that instance's `.sock.ro` socket. Each
attachment addresses one instance. QE Fleet's F1 FLEET lists every
instance the collector observes, which is a different mechanism — it
reads files, not sockets.

## Compare records without mixing account and strategy equity

`status` reads each daemon through `qe_daemon.DaemonClient`, checks strategy
and account identity, and reports **strategy-book USD equity and cash** from
`book()`. Account NAV is never the denominator. `collect` appends the complete
snapshot to `/datapool/.../observations/comparison.sqlite3`, protected against
UPDATE/DELETE, and atomically publishes `observations/status.json`.

Optional `shadow_sources` entries have `id`, `status` and `ledger` absolute
paths pointing to a risk-weighting observer. All virtual books are imported
read-only with their phase/cost IDs intact. Their last completed daily marks
and synthetic interest are labelled separately from broker-paper snapshots.
Unavailable or stale sources are visible; no missing return is filled with 0.

The initial snapshot is not an investment return since original capital.
Comparisons must use a common forward start/end and matched phase/cost.
Do not compare the incumbent's accumulated wealth with a new flat $25k book,
combine 20 phases into one selected winner, or treat reference fills as broker
execution. The collector deliberately does not manufacture a daily Sharpe or
CAGR from five-minute/incomplete observations. Paper interest stays excluded.

`compare --manifest ...` publishes `observations/comparison.json`: period return
and sampled drawdown using only common valid observations. Broker-paper
instances form one group; shadow arms are compared only within matching
phase/cost groups and completed mark dates. With fewer than two common points
the group is pending. No CAGR, Sharpe or promotion decision is inferred.

## Frozen inverse-vol candidate

`research/deploy/it-invvol-half-paper.qe` preserves Stage 3/4's score, IT30
universe order, top10/exit10, rebalance20, return-vol60 weighting, gross0.5,
max_weight2/30, noncompounding $25k, whole shares and next-open schedules.
Its research cost fields retain 10bp commission, 3bp slippage and $1/order;
the live ledger records actual broker fees rather than substituting those
assumptions for commission reports.
The account is deliberately `UNCONFIGURED` and port4004 is only a placeholder;
native validation passing does not mean that account/Gateway exists.

Its $20k gross / $3334 per-name caps allow room above initial $12.5k / $1666.67
entry budgets. The $2500 daily-loss cap is strategy-scoped. These additional
live protections were not simulated in the historical results and can alter
execution if they trip; journal every rejection/trip during the paper trial.
They do not change the frozen research allocator. At least61 warmup bars are
required, and completed-close decisions must execute at the next session open.
Current paper runtime requests delayed data; paper fills do not establish
real-time executable spreads or real-money slippage. Review that separately
before any funded promotion.

The existing shadow and its minimum60-session research review remain in force.
An IBKR paper pilot tests execution plumbing; it does not itself waive that
protocol or establish the return target.
