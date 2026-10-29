# IBKR paper-trading verification

End-to-end smoke test against a real IB Gateway paper account: can
this machine reach the venue, read account truth, and get an order
filled. Not CI-able — it assumes IB Gateway is installed and you have
paper credentials.

Use **`qe_order_probe`** for a one-shot broker round trip with text output
that can be retained as verification evidence. This command submits an order;
run it only against the intended paper account.

Daemon cold start, warmup, watchdog, reconciliation and restart checks are in
[the daemon smoke checklist](qe-daemon-smoke.md).

## Prerequisites

- IB Gateway (recommended) **or** TWS Desktop installed.
  Gateway download: <https://www.interactivebrokers.com/en/index.php?f=16457>
- Paper account credentials (free; sign up at
  <https://www.interactivebrokers.com/en/index.php?f=46385>).
- The engine CLIs built:
  ```bash
  cmake --preset=release
  cmake --build --preset=release -j
  ```

## Phase 1 — Gateway configured and listening

1. Launch IB Gateway. Pick **Paper Trading**. Log in.
2. In Gateway: `Configure → Settings → API → Settings`.
   - Enable `ActiveX and Socket Clients`.
   - Set `Socket port` to **4002** (the default for paper Gateway).
   - Add **127.0.0.1** to `Trusted IP Addresses`.
   - Untick `Read-Only API`.

**Pass:** Gateway is listening —
`nc -z 127.0.0.1 4002 && echo OK`.

**Fail modes:**

- `Connection refused` — Gateway not running, or listening on
  another port. 4001 is *live* Gateway; getting this wrong is how a
  probe aimed at paper reaches real money, which is why
  `qe_order_probe` refuses any port but 4002 without `--submit`.
- Saturday ~01:00–05:00 ET: "Unrecognized Username or Password" is
  IBKR's maintenance window, not your credentials. The gateway never
  retries by itself.

## Phase 2 — Read account truth (submits nothing)

```bash
qe_order_probe --query
```

**Pass:**

- An account id, and it starts with `DU` — the paper prefix. A `U…`
  account here means the Gateway is logged into real money.
- Net liquidation, positions and open orders print without error.

**Fail modes:**

- Exit 4: socket connect or TWS handshake failed. Phase 1 is not
  actually done.
- Account id missing: the Gateway accepted the socket but has not
  finished its own login. Wait for it to show connected; a cold-start
  login that never completes needs a container/app restart.

## Phase 3 — The order that would be sent

```bash
qe_order_probe --dry-run --symbol INTC --side buy --qty 1
```

**Pass:** it prints the contract, side and quantity and exits 0
without opening a socket. Read the line before the next phase —
this is the last point at which nothing has been sent.

## Phase 4 — A real paper fill

```bash
qe_order_probe --symbol INTC --side buy --qty 1 --wait 30
```

This **submits a market order** on the paper account. One share of a
liquid name, during market hours.

**Pass:**

- Exit 0, and the probe prints the fill price and quantity.
- The same order appears in Gateway's `API → Orders` pane.
- `~/Library/Application Support/qe_dashboard/orders.jsonl` appends
  `submit_attempt` and `submit_accepted` rows. (That directory keeps
  the old name on purpose — see
  [Credentials security](credentials-security.md).)

**Fail modes:**

| Exit | Meaning | What to check |
|---|---|---|
| 5 | `submit_order` rejected | the broker's reason in the probe output; a `Read-Only API` tick in Gateway produces this |
| 6 | submitted, no fill within `--wait` | outside market hours, or the symbol's exchange is closed. The order may still be working — check Gateway |

Then flatten it by hand in Gateway. The probe does not sell back what
it bought, and nothing in this repo cancels or closes a position you
opened this way.

## Phase 5 — The account gate refuses a mismatch

The daemon asserts which account it expects and refuses to start
against another. That gate is what turns "logged into the wrong
account" from a silent wrong-book into a startup failure, so it is
worth seeing it fire once.

1. In your smoke `.qe`, set `live(expected_account = "DU0000000")` —
   deliberately not your account.
2. Start the daemon.

**Pass:** it refuses to start and names both accounts — the one it
expected and the one it found. Nothing connects to the venue.

3. Put your real paper account back.

**Pass:** it starts.

## Recording results

```bash
# Event variety in the order log
tail ~/Library/Application\ Support/qe_dashboard/orders.jsonl

# Counts by kind
grep -o '"kind":"[^"]*"' \
    ~/Library/Application\ Support/qe_dashboard/orders.jsonl \
    | sort | uniq -c
```

Capture that, the probe's own output, and any failures alongside your
changes before shipping live-broker work.

## What this page no longer verifies

Stated rather than quietly dropped, because each one used to be a
pass criterion here:

- **A resting limit order submitted and cancelled from a GUI.** There
  is no GUI that submits. Place and pull resting orders in TWS / IB
  Gateway if you want to exercise that path at the venue.
- **The kill switch blocking a new submission while leaving a working
  order alone.** This is the daemon's switch now, and it is covered
  by a test (`kill: a client's encoded frame trips the daemon's real
  switch`) plus
  [`qe-daemon-smoke.md`](qe-daemon-smoke.md#control-socket-round-trip).
- **A broker-drift modal.** Reconciliation is the daemon's, is
  market-hours-only, and reports through `qe_daemon --status` and
  QE Fleet's F2 INSTANCE. See
  [`qe-daemon-smoke.md`](qe-daemon-smoke.md#verifying-broker-reconcile).

## See also

- [`qe-daemon-smoke.md`](qe-daemon-smoke.md) — the daemon's own
  end-to-end checklist.
- [`ibkr-connectivity.md`](ibkr-connectivity.md) — Gateway setup in
  detail, ports, client ids, common errors.
- [`live-trading-safety.md`](live-trading-safety.md) — what stands
  between a config and an order.
