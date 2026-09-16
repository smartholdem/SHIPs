```
  SHIP: 4
  Title: Web4 Peer-to-Peer Layer on Iroh (Gossip + RPC)
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Networking
  Created: 2023-10-06
  Last Update: 2026-06-19
```

## Abstract

This SHIP specifies the next-generation transport of SmartHoldem nodes: QUIC connections between node identities
(Ed25519 public keys) established directly through NAT with hole punching, falling back to relay servers only for the
handshake; block and transaction propagation over an epidemic gossip protocol; and a minimal request/response protocol
for status and block ranges. There are no central servers: any node can bootstrap from any other node it has ever seen,
and relays never see plaintext nor are required once a direct path exists.

## Motivation

Legacy nodes talk over WebSockets to IP:port pairs published in a seed list. This needs public IPs (or port forwarding),
leaks topology, propagates a block through the network in one to three seconds and depends on a handful of seed hosts.
For sub-second slots ([SHIP-39.md](SHIP-39.md)) and BFT finality ([SHIP-35.md](SHIP-35.md)) the network needs sub-100 ms fan-out, authenticated peers and
connectivity from home and mobile networks.

## Specification

### Identity and transport

- Every node has an **iroh endpoint**: an Ed25519 key pair; the public key is the node id. Connections are QUIC with TLS
  bound to the node id - peers are authenticated by construction, traffic is end-to-end encrypted.
- Path selection: direct UDP (hole punching via the peer's addresses learned from gossip / relay) -> relay fallback.
  Built-in relays `N1_RELAYS` (`relay-fsn7.sth.cx`, `relay-ru1.sth.cx`) plus optional public n0 relays and operator
  relays; relays are dumb forwarders of encrypted QUIC and are **not** trusted.
- ALPN `sth/1` identifies the SmartHoldem protocol; one endpoint serves gossip and RPC.

### Gossip topics

Topic ids are derived from the network `nethash`, so test networks never mix with mainnet.

| Topic | Message | Semantics |
|---|---|---|
| `blocks` | `Block { block }` | full block, forwarded once per id after stateless checks |
| `transactions` | `Transactions { transactions }` | transactions accepted into the sender's mempool |
| (both) | `Peers { peers, gateway }` | healthy legacy peers (reputation sharing) and the sender's legacy gateway address |
| (both) | `Delegates { delegates }` | delegates forging on the sender, each signed by the delegate key over `sha256("sth-delegate-announce" ‖ node id ‖ timestamp)` |

Gossip uses HyParView-style membership and Plumtree broadcast (iroh-gossip): each node keeps a small active view,
messages reach `N` nodes in `O(log N)` hops with duplicate suppression by message id. Measured mainnet propagation of a
block to all [Rust nodes](https://github.com/smartholdem/sth-core-rust): 60–150ms versus 1–3 s over legacy WebSockets.

### RPC

| Request | Response |
|---|---|
| `GetStatus` | `{ version, coreVersion, nethash, height, id }` |
| `GetBlocks { lastBlockHeight, limit }` | `{ blocks }` - heights in `(last, last + limit]` |

Sync prefers iroh peers: the follower asks the best-height peer for ranges of up to 400 blocks, validates the batch
(SHIP-3), and falls back to legacy REST/WebSocket only when no iroh peer is ahead.

### Bootstrap without central servers

Order of discovery: `node.yaml -> p2p.bootstrap` node ids -> peers remembered in the database from previous runs ->
`Peers` gossip -> legacy gateways announced by [Rust nodes](https://github.com/smartholdem/sth-core-rust) ([SHIP-5.md](SHIP-5.md)). A node that has ever been online can restart with
an empty bootstrap list.

### Delegate quorum

Before a delegate forges, the node checks that a `quorum_share` (default 50 %, ≥ `min_quorum_peers`) of its iroh peers
agree on the tip height and id, preventing forging on a private fork after a network partition.

## Rationale

Iroh was chosen over libp2p for a smaller footprint, first-class NAT traversal and relay design, and a QUIC stack shared
with the future light-client protocol ([SHIP-38.md](SHIP-38.md)). Gossip messages carry full blocks today; compact blocks ([SHIP-36.md](SHIP-36.md)) will cut
bandwidth once mempools are well synchronised.

## Benefits

Home and mobile nodes participate without port forwarding; block fan-out 10–20x faster; authenticated peers; no seed
server is a single point of failure; test networks are isolated by construction.

## Backwards Compatibility

Additive. [Rust nodes](https://github.com/smartholdem/sth-core-rust) speak both stacks ([SHIP-5.md](SHIP-5.md)); legacy nodes see Rust nodes as ordinary WebSocket peers.

## Reference Implementation

`src/p2p_iroh/{mod,gossip,rpc,proto,peers,delegates}.rs`; tests `tests/p2p_live.rs`, `tests/legacy_server.rs`.
