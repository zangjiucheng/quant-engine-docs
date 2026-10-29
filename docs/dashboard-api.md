# Dashboard API (read-only) — snapshot schema

The web monitoring UI reads three endpoints from a Python aggregation
service (`python/qe_dashboard_api/`). This page is the contract between
them: what a response contains, what each timestamp means, and — most
of the page — how a response says that it does **not** know something.

!!! note "Browser API and desktop transport"

    The browser UI uses the Python HTTP service. The desktop app uses its
    Rust agent transport. `QE_DASHBOARD_*` environment variables configure
    the Python service; they remain supported names.

    This contract has **three implementations**, and that is
    deliberate: the Python service above, the Rust
    `qe-fleet-core` behind QE Fleet, and the TypeScript client in
    `web/dashboard/src/api.ts`. They are held together by
    `tests/scripts/test_dashboard_parity.py`, which compares the
    Python and Rust screen-by-screen and fails on any field one has
    and the other does not.

The service is read-only by construction. It serves a fixed list of
screens, it holds a fixed list of daemon read verbs server-side, and it
accepts no verb, path or socket from a caller. The enforcement that
matters is still in the daemon (`ReadOnlyControlHandler` allow-lists
read verbs by name, per connection); this service's own list is a
second, weaker fence on the near side and is not a security property.

!!! warning "The service has no authentication of its own"
    It binds loopback by default and refuses to pretend otherwise. The
    pages it serves carry account equity, positions and order history,
    so anything that exposes it beyond loopback needs a front door that
    was verified, not assumed.

## The one rule

**An absence is never representable as a measurement.** Every scalar
and every block carries its own availability:

| `availability` | Meaning | `value` / `data` |
| --- | --- | --- |
| `ok` | Measured, and inside this block's freshness budget | present |
| `stale` | Measured, but older than the budget | present, with `age_s` |
| `unknown` | **Not measured.** `note` says why nobody has the number | **absent from the JSON** |
| `error` | Asked, and it failed. `error` says how | **absent from the JSON** |

`unknown` and `error` omit the value key entirely rather than sending
`null`. A client doing `row.equity.value.toFixed(2)` throws where the
gap is, instead of rendering `0.00` next to a strategy that is actually
unreachable.

An empty answer is not an absence. `positions: {}` with
`availability: "ok"` means the book is flat and somebody checked.

```json
{
  "value": 26051.32,
  "availability": "ok",
  "source": "fleet:status.json:incumbent",
  "as_of_ns": 1792775147569270016,
  "as_of_session": null,
  "unit": "USD",
  "note": null
}
```

## Three kinds of time, which are not interchangeable

| Field | Means |
| --- | --- |
| `as_of_ns` | When the data was true of the world. Nanoseconds since the epoch, the engine's unit. |
| `as_of_session` | A *completed trading session*, `YYYY-MM-DD`, for a number whose basis is a daily mark rather than an instant. |
| `collected_at_ns` | When the service asked. **Never** a substitute for either of the above. |
| `age_s` | Derived: `now - as_of_ns`. `null` when the block is session-dated — a daily mark has no second-resolution age, and "0 s old" would be a lie. |

A shadow book makes the distinction concrete: its equity is a mark from
the last completed session (`as_of_session`), while its positions come
from a checkpoint that may be hours newer (`as_of_ns`). One timestamp
for the pair would be wrong about one of them.

One HTTP response time is not the market time of everything in the
response. Browser refresh is not data age.

## Entities, which must not be merged

| Field | Is |
| --- | --- |
| `instance_id` | A stable running identity. Renaming a page title does not change it. |
| `strategy_id` | The strategy and config version. One strategy may have several independent instances. |
| `account_id` | A broker account. **Masked** (`DUQ***944`) — enough to tell two apart, not enough to publish. |
| `book_kind` | What kind of evidence a book is: `broker-paper`, `live`, `shadow`, `news-paper`, `unknown`. `live` is a real-money account; `unknown` means the source word was not recognised, which is deliberately not the same as paper. |
| `comparison_group` | Which books share a window, phase and cost basis, and may therefore be shown together. |
| `transport` | A socket path. **An address, not an identity** — the same path serves a different process after a restart. |

`book_kind` values are not three flavours of one thing and **their
equity is never summed**. A broker-paper book is what a paper account
holds. A shadow book is an indicative reference computed from marks,
with no account behind it — one shadow source can produce a hundred
books and zero processes. A news-paper book is the analysis service's
fixed-notional observation, which has never placed an order.

## Endpoints

All `GET`. All `Cache-Control: no-store`.

| Endpoint | Answers |
| --- | --- |
| `/api/v1/overview` | What is running, and where attention is needed |
| `/api/v1/instances/{instance_id}` | What one instance is doing now |
| `/api/v1/comparison` | Which results may be put side by side |
| `/api/v1/meta` | Schema version, available blocks, and **what this service cannot verify** |
| `/api/v1/healthz` | Liveness of *this process only* — not the daemon, not the feed, not the account |

`instance_id` may contain slashes (`risk-weighting/control/0/base`); it
is matched against the registry and the collector's rows, and never
reaches a filesystem.

### Envelope

```json
{
  "schema_version": 1,
  "generated_at_ns": 1792775167569270016,
  "blocks": { "<name>": { "availability": "...", "source": "...", "...": "..." } },
  "...": "screen-specific keys"
}
```

Added fields do not bump `schema_version`; a client must tolerate
fields it does not know. The version bumps when a field changes meaning
or disappears.

### Overview

`summary` has **three buckets, never two**:

```json
{
  "instances": { "value": 8,  "availability": "ok", "...": "..." },
  "healthy":   { "value": 5,  "availability": "ok", "...": "..." },
  "unhealthy": { "value": 1,  "availability": "ok", "...": "..." },
  "unknown":   { "value": 2,  "availability": "ok", "...": "..." }
}
```

`healthy` + `unhealthy` do not have to equal `instances`. The remainder
is `unknown` — books whose health nobody measured, because the daemon
refused the connection or the source failed wholesale. Filing those
under either of the other two buckets is the defect this shape exists
to prevent.

When the collector's own output cannot be read, **every count is
`unknown`**. `{"instances": 0, "healthy": 0}` is the shape of a quiet
Sunday and must never be the shape of a monitoring layer that has lost
sight of a live daemon. A stale artifact yields counts marked `stale`,
not recomputed ones: they are exactly as old as the file they came from.

### Instance detail

Blocks come from the daemon's read verbs (`status`, `positions`,
`orders`, `equity`, `equity_history`, `book`, `baseline`,
`reconcile_status`, `eval_history`, `eval_alerts`,
`scheduled_orders`, `log_tail`) plus the fleet's observation row.

`collection` describes the pass itself:

```json
{
  "window_ns": [1792775167000000000, 1792775167310000000],
  "complete": false,
  "identity": { "pid": 4181, "strategy_id": "...", "counters_since_ns": 1792700000000000000 },
  "identity_changed": []
}
```

Several RPCs are not one atomic snapshot. The service reads `status`
first and again last; if `pid`, `counters_since_ns`, `strategy_id`,
`account_id`, `config_path`, `version` or `host` moved in between,
`identity_changed` names the fields and **every other block becomes
`unknown`**. An orders list from the process that just died beside an
account block from the one that replaced it is indistinguishable, on a
page, from a working daemon.

`complete: false` means a block was not measured. It is not a claim
about the daemon's health — an incomplete observation and an unhealthy
instance get different words.

`evidence` carries three things the service knows it cannot prove:

* **`write_boundary`** — always `unknown`. The daemon's status payload
  has no `read_only` field, so a client cannot tell which endpoint it
  is attached to without attempting a write, which this service never
  does. Rendered as *unverified*, never as a green read-only tick.
* **`config`** — the hash the manifest pins, the path the process
  reports loading, and `process_proves_content: false`. Three different
  kinds of evidence, deliberately not merged: hashing a file on disk
  does not prove what a running process loaded.
* **`market_data`** — the IBKR market data type the venue *granted*,
  with its meaning spelled out. Type `3` is delayed by about fifteen
  minutes, which means an evaluation stamped 16:00 ranked prices from
  roughly 15:45.

### Comparison

Each group carries the collector's own numbers and an explicit
eligibility verdict:

```json
{
  "comparison_group": "broker-paper",
  "members": ["incumbent"],
  "common_observations": 24,
  "status": "monitoring_only",
  "eligibility": {
    "comparable": false,
    "reasons": ["a single member (incumbent) is not a comparison — it is one book's own history"]
  },
  "strategies": { "incumbent": { "period_return": 0.0, "sampled_max_drawdown": 0.0 } },
  "basis": "zero interest; common valid observations per group; sampled drawdown only; no CAGR/Sharpe or promotion verdict"
}
```

Only `period_return` and `sampled_max_drawdown` ever appear, because
those are the only two the collector measures. **Nothing is annualized.**
A five-minute snapshot series does not carry a CAGR or a Sharpe ratio,
and a group with fewer than two common valid observations has no return
at all — it is `pending`, with a reason that says so.

`comparable: false` has explicit reasons. A single-member group is the
common case today: there is one paper account, so the broker group has
one member and showing it as a ranking produces a winner out of a
one-horse race.

## Configuration

Server-side only, from the environment at startup. No request
influences any of it.

| Variable | Default |
| --- | --- |
| `QE_DASHBOARD_FLEET_ROOT` | `/datapool/qe-data/strategy-fleet` |
| `QE_DASHBOARD_MANIFEST` | `<root>/fleet.json` |
| `QE_DASHBOARD_STATUS` | `<root>/observations/status.json` |
| `QE_DASHBOARD_COMPARISON` | `<root>/observations/comparison.json` |
| `QE_DASHBOARD_POLL_LIVE` | `1` — set `0` to serve collector artifacts only |
| `QE_DASHBOARD_TIMEOUT_S` | `5` per daemon request |
| `QE_DASHBOARD_CACHE_TTL_S` | `3` — ten open tabs are one poll |
| `QE_DASHBOARD_MAX_WORKERS` | `4` — bounded; a browser cannot create threads |
| `QE_DASHBOARD_DEV_ORIGINS` | empty; an explicit list for the Vite dev server, never a wildcard |

```bash
python3 -m venv .venv-dashboard-api
.venv-dashboard-api/bin/pip install -r requirements-dashboard-api.txt
# The launcher re-execs into .venv-dashboard-api when it exists.
scripts/qe-dashboard-api.py \
    --fleet-root /datapool/qe-data/strategy-fleet \
    --dev-origins http://localhost:5173
```

An endpoint that fails goes into backoff, so a socket known to be down
is not reconnected once per block per open tab. The first block pays
one deadline; the rest of the pass is refused from the penalty table
without a connect.

## Layering

Everything except the HTTP edge is standard library. `contract`,
`registry`, `fleet_source`, `daemon_source` and `aggregator` import
nothing outside Python's stdlib and `qe_daemon`, which is why
`tests/scripts/test_dashboard_api.py` is a plain `ctest` entry needing
only an interpreter. Only `app.py` needs
`requirements-dashboard-api.txt`. A dependency added to reach the
other modules means the layering has slipped.
