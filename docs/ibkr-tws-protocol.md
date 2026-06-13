# IBKR TWS wire-protocol notes

Reference notes for the subset of the TWS API we implement in
`IbkrBroker`. Companion to
[`docs/ibkr-connectivity.md`](ibkr-connectivity.md) (which covers
the *user-side* setup) and
[`docs/live-trading-safety.md`](live-trading-safety.md) (which
covers what we do with the connection).

## Why we don't vendor IBKR's official TWSAPI

IBKR publishes a C++ client (`TwsApi/source/cppclient/client/`)
that decodes the protocol. We're not vendoring it. Reasons:

- **License.** The TWS API license forbids redistribution without
  written permission. Checking it into this repo is the wrong move.
- **Size.** ~50 generated `.h` / `.cpp` files, plus a build system
  that doesn't fit CMake without invasive surgery.
- **API mismatch.** It assumes asynchronous callback dispatch via
  `EWrapper`. Our `IBroker` interface is synchronous request /
  response (`qe::Result<T>`). Adapting their callback model means
  wrapping every method in a "send, block on condvar, unblock from
  callback" pattern — which we can write directly against the
  wire protocol with less ceremony.
- **Scope.** We need ~10 message types. Their client supports
  hundreds.

So: hand-rolled minimal client. Wire-protocol notes follow.

## Connection layer

TWS / IB Gateway listens on a TCP port (see connectivity doc).
The handshake is:

1. Client opens TCP connection.
2. Client sends an **API version banner**:
   `API\0` followed by a 4-byte big-endian length prefix and the
   string `"v<min>..<max>"` (e.g. `"v100..151"`).
3. Server responds with `<server-version>\0<connection-time>\0`
   (newline-separated ASCII).
4. Client sends a `START_API` message (id 71) containing client id
   + optional capabilities.
5. Server may send back account list (`managedAccts`),
   next-valid-order-id (`nextValidId`), then is ready for requests.

All messages after the banner use the same framing:

```
+----+----+----+----+--------+
| 4-byte BE length | payload |
+----+----+----+----+--------+
```

Payload is ASCII fields separated by `\0` (null bytes), ending
with a trailing `\0`. Every field is a string; the protocol does
not distinguish ints from strings at the wire level — both sides
agree by position.

Each message starts with a numeric `msg_id` and (for most
incoming messages) a `version` field.

## Message types we need

| Direction | id  | Name                  | Why                           |
|-----------|-----|-----------------------|-------------------------------|
| → server  | 71  | `START_API`           | Handshake completion          |
| → server  | 6   | `REQ_ACCOUNT_DATA`    | Subscribe to account snapshot |
| → server  | 61  | `REQ_POSITIONS`       | Position list                 |
| → server  | 64  | `CANCEL_POSITIONS`    | Stop position stream          |
| → server  | 3   | `PLACE_ORDER`         | Submit an order               |
| → server  | 4   | `CANCEL_ORDER`        | Cancel by orderId             |
| → server  | 5   | `REQ_OPEN_ORDERS`     | One-shot open-order snapshot  |
| → server  | 16  | `REQ_ALL_OPEN_ORDERS` | Reconcile snapshot            |
| → server  | 1   | `REQ_MKT_DATA`        | Live tick subscription (daemon quote source) |
| → server  | 2   | `CANCEL_MKT_DATA`     | Unsubscribe / re-arm after a flap |
| → server  | 59  | `REQ_MARKET_DATA_TYPE`| Delayed-vs-realtime selection |
| → server  | 20  | `REQ_HISTORICAL_DATA` | Warmup bar fetch              |
| → server  | 25  | `CANCEL_HISTORICAL_DATA` | Abort a warmup fetch       |
| ← server  | 1   | `TICK_PRICE`          | Streaming tick prices         |
| ← server  | 2   | `TICK_SIZE`           | Streaming tick sizes          |
| ← server  | 17  | `HISTORICAL_DATA`     | Warmup bars response          |
| ← server  | 5   | `OPEN_ORDER`          | Open order row (one per)      |
| ← server  | 53  | `OPEN_ORDER_END`      | End of open-order batch       |
| ← server  | 3   | `ORDER_STATUS`        | Order state transition        |
| ← server  | 11  | `EXEC_DETAILS`        | Fill notification             |
| ← server  | 15  | `MANAGED_ACCTS`       | Account list at handshake     |
| ← server  | 61  | `POSITION_DATA`       | Position row                  |
| ← server  | 62  | `POSITION_END`        | End of position batch         |
| ← server  | 6   | `ACCT_VALUE`          | Account field key/value       |
| ← server  | 9   | `NEXT_VALID_ID`       | First usable orderId          |
| ← server  | 4   | `ERR_MSG`             | Error / warning               |

That's it. The daemon (`qe_daemon`) uses the market-data +
historical ids for its live quote source and warmup
(EPIC-62); dashboard charts and options-chain queries stay on
Yahoo / Alpaca — we don't double up on IBKR for those.

## Order ids

IBKR's orderId is a **monotonically-increasing per-client integer
the client picks**, not server-assigned. The server returns the
next-valid starting id during handshake; we use that as our seed
and increment per submission. Our `client_order_id`
(idempotency key) is sent in the order's `orderRef` field — IBKR
echoes it back in `OPEN_ORDER` rows.

## PlaceOrder frame is all-or-nothing (server v151)

The protocol has **no field tags** — the server reads PlaceOrder
fields strictly by position, with the set of expected fields
determined by the negotiated server version. There is no
"optional trailing fields" notion: at v151 the frame is ~120
fields long and **every one must be present**, even the
deprecated ones (`eTradeOnly`, `firmQuoteOnly`, `nbboPriceCap`)
and the blocks we never use (volatility, scale, hedge, MIFID-II —
all sent as UNSET/empty).

We learned this the hard way: an earlier encoder stopped after
the shortSale block, and IBKR's parser silently consumed the
*next* order's bytes as the remainder of the first frame, then
raised **error 320 "Unable to parse field"** — surfacing as a
daemon that submitted exactly one order and then went dark.
`encode_place_order` now mirrors `ibapi/client.py::placeOrder`
field-for-field at v151. Conditional blocks gated on server
versions **above** 151 (DURATION @158, MANUAL_ORDER_TIME @169,
CUSTOMER_ACCOUNT @183, ...) are omitted; if `KMaxApiVersion`
ever bumps past 151, those fields must be appended at the
matching positions.

## Order acks & timeouts

`submit_order` / `cancel_order` block on a condvar until IBKR
sends a non-PendingNew `ORDER_STATUS` (or terminal state for
cancel), an `ERR_MSG` for the id, or the connection drops. That
wait is bounded by a **10 s ack timeout**: outside regular
trading hours, paper accounts silently queue the order for the
next session open and never send the state transition — without
the bound, an after-hours submit hung the calling thread forever.
On timeout the call returns `IoError`; the order **was** written
to the socket, so the intent stays recorded and broker reconcile
at the next restart picks up whatever IBKR actually did with it.

## Historical-data quirks

`REQ_HISTORICAL_DATA` is sent with `format_date=2` (Unix epoch
seconds). IBKR honors that for **intraday** bars only —
daily/weekly/monthly bars always come back as `"YYYYMMDD"`
strings regardless of the flag. The decoder special-cases the
8-digit shape and converts it to the UTC session close (20:00Z =
16:00 EDT; strict-EST winter dates land at 19:00Z, acceptable
until a market-calendar lookup is wired through). Without this,
`20260414` was digit-accumulated into an epoch in August 1970,
which broke `BarHistory` ordering against live bars and prevented
eval-backfill from matching its close target.

## Error handling

`ERR_MSG` (id 4) carries `(reqId, errorCode, errorMessage)`.
Numeric error codes 100–449 are warnings, 1100–2110 are connection
events, 2000+ are warnings/notices, 10000+ are order-side rejects.
Our adapter maps:

- 1100 / 1101 / 1102 → connection state change (badge OFFLINE)
- 200, 201, 202, 203 → order reject (`Result<>` Error in
  `submit_order`)
- 502, 503, 504 → bad client connection (`IoError`)
- Everything else → log + ignore (or surface to UI via the
  reconcile modal)

## Threading

Read side runs on a dedicated worker thread that:

1. Reads 4-byte length, then payload, indefinitely.
2. Decodes msg_id + version + fields, dispatches to a per-id
   handler.
3. Handlers either update shared state (open orders, positions,
   account) or signal a request's pending condvar.

Write side is mutex-guarded — any caller can `submit_order`,
`cancel_order`, etc. concurrently; sends are serialized.

Request/response correlation: server only knows requestIds we
gave it. We map requestId → pending future internally.

## Reconnect

If the read thread sees EOF or a socket error:

1. Mark connection state OFFLINE.
2. Cancel all pending request futures with `IoError`.
3. Don't auto-reconnect. Per safety doc: "We never auto-restart
   TWS." The user re-launches Gateway and (eventually) the
   dashboard's `Reconnect IBKR` action triggers a new
   `IbkrBroker::connect()`.

## Test posture

Pure protocol — no network — tests cover:

- Banner roundtrip: encode v100..151, decode server version.
- Message framing: length-prefixed null-separated fields encode +
  decode roundtrip for each message type we use.
- Field encoding: int / double / bool / string serializers.
- Error-code → ErrorCode mapping table.

Network tests live behind the same `ALPACA_PAPER_LIVE_TESTS` gate
pattern (`IBKR_PAPER_LIVE_TESTS=1` env var) so CI doesn't try to
hit a Gateway that isn't there.
