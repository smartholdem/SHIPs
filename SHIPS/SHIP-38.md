```
  SHIP: 38
  Title: Light Clients - Verifiable State Commitments
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-15
  Last Update: 2026-09-03
```

## Abstract

Let wallets and mobile apps verify balances, tokens and sObjects **without trusting a REST node**: block headers commit to a
Merkle root of the whole state (`stateRoot`), full nodes serve compact **proofs** for any key over Iroh ([SHIP-4.md](SHIP-4.md)), and light
clients follow headers plus finality certificates ([SHIP-35.md](SHIP-35.md)) to know which root is authoritative.

## Motivation

Today a wallet asks one node `GET /api/wallets/:addr` and believes the answer. Iroh already lets phones connect to any node
directly; adding state commitments turns that connection into cryptographic assurance and enables serverless wallets,
pruned nodes ([SHIP-31.md](SHIP-31.md)) and cross-chain bridges that verify SmartHoldem state.

## Specification

### State commitment

- Binary Merkle tree (SHA-256 domain-separated; Poseidon2 variant reserved for [SHIP-29.md](SHIP-29.md)) over the sorted state key space:
  `w:` (wallet records serialised canonically), `tk:`, `en:`, `eo:`, `mk:`, `lk:`, `pqk:`, `dv:`.
- Recomputed incrementally per block from the same deltas that update Sled ([SHIP-3.md](SHIP-3.md)) - cost O(changed keys x log n).
- `stateRoot` in the block v1 header extension (with [SHIP-22.md](SHIP-22.md) / [SHIP-35.md](SHIP-35.md) fields); a mismatch is a validation failure.

### Proofs

RPC `GetProof { height, keys[] }` -> `{ stateRoot, proofs: [(key, value?, siblings[])] }` (absence proofs included). Typical
proof ≈ 1KB per key. `GetHeaders { from, count }` returns headers + finality certificates.

### Light client protocol

1. Start from a hard-coded checkpoint (height, blockId, delegate set) shipped with the wallet.
2. Sync headers; accept a header if it is signed by a delegate of the current active set and, when available, backed by a
   finality certificate (≥ 15 votes). Delegate set changes are read from `dv:` proofs at round boundaries.
3. Query proofs for the wallet's keys against the latest final `stateRoot`; render only verified values.
4. Submit transactions to any node (or gossip them directly); watch inclusion via proofs of `wt:` list entries.

### Bandwidth

Header ≈ 200B (+2.4KB with PQ block signatures) per 8s; a wallet syncing after a week downloads ~15MB of headers
([SHIP-30.md](SHIP-30.md) epoch compression applies to header ranges too).

## Rationale

State roots are the one consensus addition needed by pruning, light clients, bridges and future ZK systems; committing to the
existing Sled layout avoids a second state representation.

## Backwards Compatibility

Block v1 (Rust-core-only network); until then the API stays the trusted path.

## Reference Implementation

Not started; `src/storage.rs` (incremental Merkle), `src/p2p_iroh/rpc.rs` (`GetProof`, `GetHeaders`), `sth-light` crate.
