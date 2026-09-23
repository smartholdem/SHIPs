```
  SHIP: 44
  Title: Wallet Stream API - REST over iroh, Push Subscriptions, MessagePack Frames and the WebSocket Bridge
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Networking / Interface
  Created: 2026-09-19
  Last Update: 2026-09-19
  Requires: 4, 42, 43
```

## Abstract

Wallets and dApps today poll REST (`/api/wallets/:id`, `/api/transactions/:id`) every 3–8 seconds. One wallet is ~17 000
requests a day; a thousand wallets are ~200 HTTP requests per second that mostly answer "nothing changed". SHIP-44 moves
the client side to the node's iroh endpoint:

1. **`netfory/api/1`** - every existing REST endpoint over an iroh QUIC bi-stream. The request is the netfory-provider
   `MeshPacket`; the response is the node's JSON plus the signed `pdata` envelope. Existing `api://<EndpointId>/...`
   clients work against `sth-core` directly, without a separate netfory-provider process.
2. **`sth/events/1`** - one long-lived bi-stream per wallet. The wallet subscribes to addresses and topics; the node pushes
   blocks, confirmed transactions, balances, mempool arrivals and SHIP-43 order updates as they happen and replays what a
   reconnecting wallet missed.

Expected effect: 95–99 % fewer client requests on Service Nodes, sub-second UI updates, no HTTP layer (nginx/TLS certs)
needed for the wallet path. QUIC + iroh give NAT traversal, relay fallback, TLS 1.3 and mutual EndpointId authentication.

## Motivation

* Polling is wasted work: the answer is "unchanged" >99 % of the time, yet each request costs a TCP/TLS handshake or a
  keep-alive slot, request parsing, a sled lookup and JSON rendering.
* Polling has latency of half the interval on average; incoming payments feel slow.
* Browser-style REST needs a public HTTP front (nginx, certificates, CORS); Service Nodes should be reachable by
  EndpointId alone (SHIP-42 `iroh://<EndpointId>?roles=api`).
* The Netfory ecosystem already speaks `api://` (netfory-provider). Supporting the same wire format natively removes a
  hop and a process.

## Specification

### Transport

* iroh endpoint of the node (same key as gossip / `sth/rpc/1`). The node's **EndpointId is the Ed25519 public key** that
  signs `pdata`, so a client that dials `EndpointId` and verifies `pdata.sig` needs no other trust anchor.
* ALPNs: `netfory/api/1` (length-prefixed, primary), `netfory/api/0` (raw JSON + EOF, legacy clients), `sth/api/1` (alias
  of v1), `sth/events/1` (subscriptions).
* Framing (v1, events): `[u32 little-endian length][payload]`, payload = UTF-8 JSON. Limits: request ≤ 2 MiB, response
  ≤ 8 MiB, event command ≤ 64 KiB.

### `netfory/api/1` - request / response

One request per bi-stream; the connection is kept and a new bi-stream is opened per request.

Request (`MeshPacket`, netfory-provider format):

```json
{ "target_provider": "", "method": "GET", "path": "/api/wallets/SXyz...", "body": "<base64>",
  "headers": { "content-type": "application/json" }, "ttl": 5, "request_id": "a1b2c3..." }
```

* `path` - the REST path as served on `127.0.0.1:4003` (`/api/...`). Query strings are allowed. A path without the `/api`
  prefix (a provider client parsing `api://<id>/api/blocks` as name `api` + path `/blocks`) is mapped onto `/api/...`.
* `body` - base64 of the HTTP body (POST `/api/transactions`), empty for GET.
* `target_provider` - ignored for routing (a node serves itself), echoed back as `pdata.name` (`sth-core` when empty).
  The node's address in provider notation is therefore `api://<EndpointId>/<any name>`; `sth-core iroh-id --url` prints
  `api://<EndpointId>/sth-core`, `/api/node/status -> irohWalletApi.url` shows the same.
* `method: "WS"` (provider WebSocket tunnel) is answered 400 - use `sth/events/1`.

The node builds an HTTP request from the packet and runs it through its axum router **in-process** (no socket); handlers,
JSON and error envelopes are byte-identical to HTTP.

Response = node JSON with `pdata` merged in (non-object bodies are wrapped as `{ "pdata": … }`):

```json
{ "data": { ... },
  "pdata": { "v": 1, "node_id": "<EndpointId hex>", "name": "sth-core", "status": 200, "signed_at": 1750000000,
             "relayed": false, "alg": "ed25519", "sig": "<hex 64 bytes>", "body_b64": "<base64 raw body>" } }
```

Signature: Ed25519 by the node key over `node_id|status|signed_at|` + raw body bytes (`body_b64` decoded). Verify with the
dialled EndpointId; then use `body_b64` as the authoritative body.

Errors are the legacy envelope `{ statusCode, error, message }` with the same status in `pdata.status`:
503 wallet API disabled on this node, 429 rate limit for this EndpointId, 400 malformed packet.

### `sth/events/1` - subscriptions

The client opens one bi-stream and sends commands; the node answers with frames on the same stream. The node sends
`hello` as the first frame once it sees the stream (send any command, e.g. `ping`, to open it).

Commands (client -> node):

| type          | fields                                                                                                                                                  | effect                                                                                                                                                           |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `subscribe`   | `addresses: [..]`, `topics: [blocks, wallet, mempool, dex]` (default `wallet, mempool`), `tokens: [tokenId..]` (dex filter, empty = all), `sinceHeight` | replaces the subscription; replays missed confirmed transactions (≤ 200), then a `balance` per address, then `subscribed`                                        |
| `unsubscribe` | -                                                                                                                                                       | clears everything, answers `subscribed` with empty lists                                                                                                         |
| `ping`        | -                                                                                                                                                       | `pong { height, time }`; also resets the idle timer                                                                                                              |
| `encoding`    | `encoding: json \| msgpack`                                                                                                                             | switches the frame codec for the rest of the session; answered with `encoding { encoding }` already in the new codec. Any other command may carry `encoding` too |

Frame codec: a session starts in JSON; `hello.encodings` lists `["json", "msgpack"]`. After `{ "type": "encoding", "encoding": "msgpack" }`
the node emits MessagePack maps (same keys / values as the JSON objects, numbers as integers, amounts still strings) and accepts
commands in either codec (auto-detected: a payload starting with `{` is JSON). The `[u32 LE len]` framing is unchanged.
MessagePack cuts push traffic by ~25–35 % - recommended for mobile wallets.

Frames (node -> client), all carry `"type"`:

| type         | when                                                                  | fields                                                                                                                                           |
|--------------|-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| `hello`      | stream opened                                                         | `nodeId, nethash, height, coreVersion, topics, limits { maxAddresses, idleTimeoutSecs, maxReplay }`                                              |
| `subscribed` | after `subscribe`/`unsubscribe`                                       | `addresses, topics, tokens, height, replayed`                                                                                                    |
| `block`      | topic `blocks`, every applied block                                   | `block` = `/api/blocks/:id` transformed header (no transactions)                                                                                 |
| `tx`         | topic `wallet`, a confirmed transaction touching a subscribed address | `addresses` (subscribed addresses it touches), `height`, `tx` (`/api/transactions/:id` shape, `confirmations ≥ 1`), `replay: true` when replayed |
| `balance`    | topic `wallet`, after `tx` frames of a block, after subscribe         | `address`, `wallet` (`/api/wallets/:id` shape; balance `"0"` for unknown wallets)                                                                |
| `mempoolTx`  | topic `mempool`, accepted into this node's pool                       | `addresses`, `tx` (`confirmations: 0`)                                                                                                           |
| `order`      | topic `dex`, after TokenSell / TokenBuy / TokenCancel                 | `tokenId, symbol, action (sell/buy/cancel), txId, order` (`/api/tokens/:key/orders` shape)                                                       |
| `trade`      | topic `dex`, after TokenBuy                                           | `tokenId, symbol, trade { txId, orderId, buyer, seller, amount, price, total, height, timestamp }`                                               |
| `rollback`   | the tip moved down                                                    | `height` (new tip) - refresh balances / pending items                                                                                            |
| `resync`     | node dropped events (slow client) or > 200 missed transactions        | `reason` - refresh over the API                                                                                                                  |
| `pong`       | after `ping`                                                          | `height, time`                                                                                                                                   |
| `error`      | bad command                                                           | `code` (`bad_frame, unknown_command, bad_address, bad_topic, too_many_addresses, idle`), `message`                                               |

Ordering guarantees: within one block the node sends `block` -> `tx*` -> `balance*` -> `order/trade*`. Frames are not
individually signed - the QUIC connection is authenticated by the node's EndpointId (TLS 1.3); a wallet that wants a
signed statement re-reads the item over `netfory/api/1`.

Delivery is at-most-once per session. A wallet persists the last `height` it processed and reconnects with
`subscribe { sinceHeight }`; the node replays confirmed transactions from the per-wallet height index and sends fresh
balances, so nothing is lost across sleep / network changes.

### Node-side limits (node.yaml `p2p.iroh.wallet_api`)

```yaml
p2p:
  iroh:
    enabled: true
    wallet_api:
      enabled: true
      max_connections: 2000    # requests + event sessions together; sized for a 1–2 GB RAM node
      max_addresses: 64        # per event session
      rate_limit_rps: 20       # API requests / s per EndpointId, burst 40
      idle_timeout_secs: 300   # event session without ping/command is closed
```

Memory: an idle QUIC connection is ~60–100 KB (buffers + crypto state), so 2000 connections ≈ 120–200 MB. Suggested
values: 1–2GB -> 2000, 4GB -> 5000, 8GB -> 10 000. Excess connections are closed immediately with code 1; a disabled
API answers 503 (requests) / closes with code 2 (events). Stats: `/api/node/status -> data.irohWalletApi`.

Requires `api.enabled: true` (the router is shared with the local REST API; the HTTP listener itself may stay bound to
`127.0.0.1`).

### Privacy

The node learns which EndpointId watches which addresses. Wallets SHOULD use an **ephemeral iroh key per session** (or per
install) - the EndpointId is only a transport identity, nothing on chain refers to it - and MAY spread subscriptions over
several Service Nodes. Nodes MUST NOT persist the EndpointId ↔ address mapping beyond the session.

### Browser bridge (`p2p.iroh.ws_bridge`)

Browsers cannot open QUIC, so any node - usually a Service Node with the `api` role - may run the built-in bridge:

```yaml
p2p:
  iroh:
    ws_bridge:
      enabled: true
      listen: 0.0.0.0:4005      # put nginx + TLS in front for wss://
      default_node: ""          # EndpointId served when the URL names none; empty = this node
```

* `ws://host:4005/ws/events[/<EndpointId>]` - one WebSocket message = one `sth/events/1` frame (Text for JSON, Binary for
  MessagePack); the bridge adds / strips the length prefix and forwards bytes untouched.
* `ws://host:4005/ws/api[/<EndpointId>]` - one WebSocket message = one `MeshPacket` JSON, the reply message is the node's
  signed response (`pdata` verifiable in the browser against the EndpointId in the URL).
* `GET /` - bridge info (`node`, `defaultNode`, paths).

The bridge dials the target from its own ephemeral iroh endpoint (iroh refuses self-connections), so it can front the node it
runs in or any other node by EndpointId; the target node's `wallet_api` limits (rate, connections) apply per bridge, so a
public bridge is itself a Service Node responsibility ([SHIP-42.md](SHIP-42.md) `api` role).

## Rationale

* Reusing the netfory-provider wire format makes every `api://` client compatible on day one and lets a Service Node
  drop the separate provider process.
* Running requests through the axum router in-process guarantees behavioural parity with HTTP at zero maintenance cost.
* JSON frames (not a binary schema) keep the browser bridge and Vue/TS code trivial; MessagePack (`rmpv` is already a
  dependency) can be negotiated later via a `hello` capability without changing the frame layout.

## Backwards Compatibility

Additive: new ALPNs on the existing endpoint; REST, gossip and `sth/rpc/1` are unchanged. Nodes without SHIP-44 simply
refuse the ALPN and wallets fall back to REST / netfory-provider.

## Reference Implementation

Implemented in `sth-core-pq` 0.22: `src/p2p_iroh/wallet_api.rs` (both protocols, MessagePack codec, reference client
`wallet_api::client`), `src/p2p_iroh/ws_bridge.rs` (WebSocket bridge), `Storage::subscribe_chain` / `Mempool::subscribe`
event sources, config `p2p.iroh.wallet_api` / `p2p.iroh.ws_bridge`, suite `tests/iroh_wallet_api.rs` (signed REST over
iroh, push / replay / rollback, MessagePack switch, limits, bridge).

## Security Considerations

* Public data only; transaction broadcast is `POST /api/transactions` with the wallet's own signatures - no node-side
  authentication is needed. Administrative endpoints are not exposed over iroh (none exist in the REST API).
* DoS: per-EndpointId token bucket, connection cap, frame caps, idle timeout, replay cap. Event fan-out is filtered
  per session from in-memory broadcast channels (no per-client DB scans except on `subscribe`).
* A slow client that falls behind the 512-event broadcast buffer gets `resync` instead of stalling the node.
* Iroh relays only see encrypted QUIC; direct paths are preferred once hole-punching succeeds.
* The WebSocket bridge terminates TLS at nginx and dials the node from one ephemeral iroh key, so the node's per-EndpointId
  limits apply to the bridge as a whole; a public bridge is a Service Node responsibility (SHIP-42 `api` role) and SHOULD
  add its own per-IP `limit_conn` / `limit_req` (see `docs/nginx/ws-bridge.smartholdem.io.conf`).

## Future Work

* Browser bridge (WebSocket ⇄ iroh) as a Service Node mode or a standalone CLI (`sth-bridge`). - **done**: `p2p.iroh.ws_bridge`.
* MessagePack frames negotiated in `hello`. - **done**: `encoding` command.
* Signed event frames on request (`subscribe { signed: true }`).
* Service Node discovery for wallets from `GET /api/masternodes?role=api` ([SHIP-42.md](SHIP-42.md)) with health scoring.

## Copyright

This document is placed in the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
