```
  SHIP: 45
  Title: Network Autonomy Double-Forge Guard, Relay Guard, sthpq Address Book and Mainline DHT Discovery
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Networking / Protocol
  Created: 2026-09-24
  Last Update: 2026-09-24
  Requires: 4, 35, 44
```

## Abstract

SHIP-45 makes a PQ node independent of third-party network infrastructure and of operator mistakes during the
legacy -> PQ transition. It specifies four optional node modules, none of which changes consensus:

1. **Double-forge guard** (`delegate.guard`) detects a delegate passphrase loaded on two nodes at once and makes the
   PQ node *yield* its slots instead of splitting the chain.
2. **Relay guard** (`p2p.iroh.relay_guard`) removes a misbehaving relay server from the endpoint's relay map at
   runtime so the home relay migrates to the NETFORY n1 relays.
3. **sthpq address book** (`p2p.iroh.discovery`) signed dial records of PQ nodes exchanged over gossip, RPC and the
   legacy port, with Bitcoin-addrman liveness rules. Peers are found without the n0 DNS service.
4. **Mainline DHT discovery** (`p2p.iroh.discovery.mainline`) pkarr records published as BEP-44 mutable items in the
   BitTorrent Mainline DHT, the same scheme NETFORY clients use.

With all four enabled a node needs neither the public n0 relays nor `dns.iroh.link`; n0 remains an optional fallback.

## Motivation

On 2026-09-23 the mainnet split at height 11 926 030. Forensics (legacy PM2 logs 14:54–14:59 UTC) showed that the legacy
nodes never saw a competing PQ block they forged their own branch while the PQ nodes forged theirs. The trigger was a
delegate passphrase present on a legacy and a PQ node simultaneously: two valid blocks for one slot, two branches, mutual
isolation. Recovery required `rollback` and a fresh snapshot (`sth-core snapshot dump`, 0.25.17).

The same day wallets reported `incoming connection did not complete the handshake ... remote=relay euc1-1.relay.n0.iroh.link`.
The node's home relay was a public n0 relay: iroh picks the home relay by latency only, the n0 relays are rate-limited and
documented as "not for production", and nothing re-evaluated the choice once handshakes started to fail.

Both incidents share a root: the network relied on components it did not control (a second copy of a key, a public relay,
a public DNS). SHIP-45 replaces every such dependency with a module the network itself operates, and turns the remaining
third parties into fallbacks.

## Specification

### 1. Double-forge guard

```yaml
delegate:
  guard: { enabled: true, mode: yield, rounds: 2 }   # mode: yield | alert
```

Definitions. *Own key* = a delegate public key whose passphrase is loaded on this node. *Foreign block* = a block whose
`generatorPublicKey` is an own key, that entered the chain through a network intake path (legacy pull, legacy `postBlock`,
iroh gossip), whose id is not in the node's own-forged set, and whose timestamp is within `rounds × activeDelegates ×
blocktime` seconds of now. *Competing block* = a block at a height the node already has, with a different id, the same
generator, and a valid block signature.

Rules.
- A foreign block MUST be logged at ERROR (`DOUBLE FORGING RISK`) and MUST raise alert kind `double-forge`.
- A competing block MUST be logged at ERROR (`DOUBLE FORGING by <delegate>`) and MUST raise alert kind `double-forge`.
  Blocks with an invalid signature MUST be ignored (no alert): anyone can send garbage.
- In `mode: yield`, when the slot of an own key comes up at height *h* and a foreign block by that key exists with
  height ≥ *h − rounds × activeDelegates*, the node MUST skip the slot with reason `double-forge-guard: ...` and raise alert
  kind `double-forge-yield`. Forging resumes automatically when no foreign block lies inside the window.
- In `mode: alert` the node MUST NOT skip slots.
- The guard MUST never reject, roll back or reorder blocks: it only decides whether *this* node forges.

Metrics: `doubleForgeGuard { enabled, mode, rounds, yieldedSlots, yielding[], foreign[], competing[] }`.

### 2. Relay guard

```yaml
p2p.iroh:
  relay_n0: false                                   # default since 0.25.19
  relay_guard: { enabled: true, max_failures: 5, window_secs: 600 }
```

*Protected relays* = the built-in NETFORY n1 relays plus `p2p.iroh.relays`. The node counts failed incoming handshakes
(wallet protocols `netfory/api/1`, `sth/events/1`) per relay URL of the remote path. When a non-protected relay reaches
`max_failures` within `window_secs` the node MUST call `Endpoint::remove_relay(url)`, log WARN and raise alert kind
`relay-degraded`; iroh then re-selects the home relay among the remaining ones. A protected relay MUST never be removed
(WARN only). `sth-core doctor` MUST warn when a home relay matches `*.relay.n0.iroh.link`.

### 3. sthpq address book

#### 3.1 Record
```json
{ "v": 1, "id": "<EndpointId hex>", "addrs": ["<ip>:<udp port>", ...], "relay": "<https url>" | null,
  "seq": <unix seconds>, "version": "<sth-core version>", "signature": "<ed25519 hex>" }
```
`signature` = Ed25519 by the node key (EndpointId) over
`sha256("sthpq-addr-v1" ‖ nethash ‖ id ‖ seq (u64 LE) ‖ addrs sorted, joined by "," ‖ relay or "")`.

Validation (all MUST hold): `v == 1`; `len(addrs) ≤ 8`; every address is globally routable (no RFC 1918, CGNAT
100.64/10, loopback, link-local, unspecified, broadcast, documentation ranges, ULA/link-local IPv6); `now − expire ≤ seq ≤
now + 600`; signature valid; `addrs` or `relay` non-empty. For one id the record with the higher `seq` wins.

#### 3.2 Channels
| Channel | Transport                                         | Payload                                                                |
|---------|---------------------------------------------------|------------------------------------------------------------------------|
| gossip  | iroh-gossip topic `sha256("sth/<nethash>/addrs")` | `GossipMessage::Addrs { records }` own record + ≤ 16 fresh others      |
| RPC     | `sth/rpc/1` (SHIP-4)                              | `Request::GetAddrs { limit }` -> `Response::Addrs { records }` (≤ 100) |
| legacy  | port 4001 route `p2p.pq.getAddrs`                 | JSON array of records; served by PQ nodes only (`CORE_PQ` plugin)      |

A node announces on start (+15 s) and every `announce_secs` (default 1800). Nodes without the module never join the topic
and answer `Error: invalid request` to `GetAddrs`; callers MUST treat that as "reachable, no address book".

#### 3.3 Liveness (Bitcoin addrman rules)
Per entry: `firstSeen, lastSeen, lastSuccess, lastAttempt, failures, source`.
- **terrible** ⇔ (`lastSuccess == 0 ∧ failures ≥ 3 ∧ now − firstSeen > 1 d`) ∨ (`lastSuccess > 0 ∧ now − lastSuccess > expire_days`).
  Terrible entries MUST be evicted (memory and sled `ab:<id>`).
- **fresh** ⇔ `now − lastSuccess ≤ 3 d`. Only fresh records MAY be handed to other nodes.
- Crawler (every 5 min): mark iroh peers heard within 10 min as alive; probe ≤ 6 entries silent > 1 h and untried 30 min
  with `GetStatus`; pull `GetAddrs` from ≤ 3 neighbours; pull `p2p.pq.getAddrs` from ≤ 4 legacy PQ peers; evict.

Accepted records MUST be inserted into the endpoint's `MemoryLookup` so iroh dials direct addresses first.

#### 3.4 Bootstrap and stable addresses
`p2p.iroh.bootstrap` entries MAY carry a direct address: `<EndpointId>@<ip>:<udp port>`. `discovery.listen_port` pins the
endpoint's UDP port; `discovery.external_addr` adds the public `ip:port` to the endpoint (and hence to the record).
Gateway nodes SHOULD set both.

### 4. Mainline DHT discovery

`discovery.mainline: true` adds an address-lookup service backed by the BitTorrent Mainline DHT (BEP-5/BEP-44):
- key = the node's Ed25519 public key (EndpointId), `salt = None` -> DHT target `sha1(pubkey)`;
- value = pkarr `SignedPacket` (DNS packet, TXT `_iroh.<z32(EndpointId)>` with `relay=` and `addr=` entries), signed by the
  same key; `seq` = packet timestamp, the DHT keeps the newest;
- publish on address change (50 ms debounce) and every 60 min; resolve = `get_mutable_most_recent`;
- bootstrap = `router.bittorrent.com`, `dht.transmissionbt.com`, `dht.libtorrent.org`, `relay.pkarr.org` + `mainline_bootstrap`;
- **address filter:** only relay URLs and globally routable IPs are published.

This is byte-compatible with the NETFORY client stack (`iroh-mainline-address-lookup` + `n0-mainline`): a wallet that
resolves a node through the DHT and a node that resolves a peer through it read the same records.

### 5. n0-free mode
`discovery.n0_dns: false` builds the endpoint with iroh's `Minimal` preset: no n0 DNS publish/resolve, no n0 relays.
Discovery = address book + DHT; relays = n1 (+ `relays`) as fallback for NAT participants. The default is `true` until an
operator confirms `ntfry.discovery.fresh > 0` on the network. The SHIP-44 WebSocket bridge MUST reuse the node's main
endpoint for remote targets and a `Minimal`, relay-less client endpoint for the local target.

## Rationale

- *Yield, not fork-choice.* A same-height fork-choice on the PQ side was discussed and rejected for now: legacy nodes
  cannot yield, and a rule that rolls back valid blocks is a consensus change with its own failure modes. Yielding is
  strictly safer the worst case is a skipped slot.
- *Bitcoin addrman rules* are battle-tested for exactly this problem (peers discovered long ago that no longer answer).
- *Own address book first, DHT second.* The book covers PQ ↔ PQ discovery with zero third parties and sub-second latency;
  the DHT adds a path for clients and for nodes whose bootstrap list is stale.
- *Signed records instead of trusted lists*: any peer can relay a record, nobody can forge one.

## Backwards Compatibility

All modules are additive and default-on with safe behaviour. Nodes on older builds: never see the `addrs` topic, answer
`invalid request` to `GetAddrs` and `unknown route` to `p2p.pq.getAddrs` callers degrade gracefully. Existing
`node.yaml` files parse unchanged (`#[serde(default)]`). `relay_n0` changing its default to `false` only affects nodes
that did not set it explicitly.

## Reference Implementation

`sth-core` 0.25.18 – 0.25.21: `src/delegate/guard.rs`, `src/p2p_iroh/relay_guard.rs`, `src/p2p_iroh/addrbook.rs`,
`p2p_iroh::mainline_lookup`, `src/p2p_iroh/ws_bridge.rs`; storage prefix `ab:`; tests in the respective modules and
`tests/iroh_p2p.rs`. Operator guide: `docs/NETWORK-AUTONOMY_RU.md`.

## Security Considerations

- Records are signed by the node key and bound to the nethash: replay across networks and address forgery are impossible;
  a stolen node key only lets the attacker redirect *that* node's dial info, and the newest `seq` from the real node wins.
- Address spam is bounded by `max_entries` (2000), the 8-address cap per record and the public-IP filter.
- Competing-block detection verifies the block signature before alerting, so it cannot be used to spam alerts.
- The relay guard can only remove relays that are not protected; an attacker who floods failed handshakes through the
  n1 relays only produces warnings.
- The DHT reveals `EndpointId -> public ip:port`; the address filter guarantees LAN addresses of NAT nodes never leave the
  host. Nodes that must stay unlisted set `mainline: false`.

## Future Work

- Move the WebSocket bridge's browser clients to DHT resolution (NETFORY client stack) and drop n0 DNS by default.
- Immediate same-height fork-choice with deterministic tie-break reconsider once no legacy delegates remain.
- Address-book entries as input for SHIP-42 Service Node uptime proofs.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
