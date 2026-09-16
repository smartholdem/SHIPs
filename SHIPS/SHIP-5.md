```
  SHIP: 5
  Title: Dual-Stack Networking: Legacy WebSocket Bridge and Peer Discovery
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Networking
  Created: 2024-10-13
  Last Update: 2026-06-19
```

## Abstract

`sth-core` speaks the legacy peer protocol (WebSocket, port 4001) inbound and outbound in addition to iroh (SHIP-4). This
SHIP defines how the two stacks are bridged so that a mixed network stays one network: blocks and transactions received on
either side are re-broadcast on the other, legacy peers are discovered and health-scored without seed servers, and Rust
nodes announce themselves as **gateways** for legacy traffic.

## Motivation

Consensus features can only be scheduled when every active delegate runs `sth-core` ([SHIP-1.md](SHIP-1.md)). Until then the Rust node
must be a first-class citizen of the legacy network and must not partition it; afterwards the bridge keeps explorers,
exchanges and old wallets working while they migrate.

## Specification

- **Inbound**: the node serves the legacy WebSocket protocol (`p2p.legacy_port`, default 4001): `getStatus`,
  `getBlocks`, `postBlock`, `postTransactions`, `getPeers`, with the same JSON shapes and rate limits the legacy node applies.
- **Outbound**: a `PeerTable` of legacy peers with health scores (latency, height, error streak) refreshed continuously;
  peers are learned from `getPeers` of any healthy peer, from `Peers` gossip of Rust nodes and from `sync.rest_nodes`.
- **Bridge**: a block arriving over iroh is forwarded to legacy peers with `postBlock` (fan-out `relay_fanout`) and vice
  versa; transactions likewise. Duplicates are suppressed by id on both sides.
- **Gateway announce**: a Rust node reachable on its legacy port publishes `Peers { gateway: "ip:4001" }` over gossip so that
  newcomers on either stack find a bridge quickly; the metrics page ([SHIP-8.md](SHIP-8.md)) uses gateway addresses to tell which
  legacy-visible peers are actually Rust nodes.
- **Legacy evidence**: for every delegate the node records through which stack its recent blocks arrived; a delegate whose
  blocks are only ever seen from non-gateway legacy peers is inferred to run legacy software (`implementation: legacy`).

## Rationale

Bridging at the message level (rather than proxying connections) keeps each stack's validation and rate limits intact and
lets the iroh side become the primary path as soon as enough Rust nodes exist, with no flag day.

## Backwards Compatibility

Full; legacy nodes need no change.

## Reference Implementation

`src/p2p_legacy/{server,peers,health,...}.rs`, `src/intake.rs`; tests `tests/legacy_server.rs`, `tests/p2p_live.rs`.
