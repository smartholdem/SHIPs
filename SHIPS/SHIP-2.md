```
  SHIP: 2
  Title: Rust Core Node (sth-core)
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2025-09-08
  Last Update: 2026-06-19
```

## Abstract

This SHIP specifies `sth-core`, a full reimplementation of the SmartHoldem node in Rust that is byte-for-byte compatible with
the historical chain (block and transaction hashing, signatures, state transitions) and with the legacy REST API, while
replacing the storage engine, the networking stack and the execution pipeline. It is the platform on which every later SHIP
is built.

## Motivation

The legacy node (TypeScript, PostgreSQL) needs ~4 GB of RAM, tens of gigabytes of disk, several minutes to start and has a
throughput ceiling far below what the 8-second DPoS slot allows. Its dependency tree is large and partly unmaintained,
which makes security fixes and new consensus features slow and risky. A memory-safe, single-binary node with deterministic
performance lets the network evolve (post-quantum signatures, native tokens, sub-second slots) without inheriting that debt.

## Specification

### Compatibility invariants

1. Block id, transaction id and signing hashes are computed exactly as the legacy core does (SHIP-11); the whole mainnet
   history validates from block 1 with identical state (balances, votes, delegate ranks, second keys, locks).
2. The REST API (`/api/*`) keeps the legacy response shapes; new fields are additive.
3. Consensus parameters come from `network.json` / `milestones.json` / `exceptions.json`, deep-merged by height;
   unknown historical keys are preserved.

### Architecture

| Module | Responsibility |
|---|---|
| `sync` / `follow` | chain following, batch validation of blocks, fork detection and automatic rollback |
| `rules` | stateless and wallet-aware transaction rules (fees, nonces, balances, second signatures, sObjects, tokens, PQ) |
| `storage` | Sled key-value state with undo log (SHIP-3) |
| `p2p_iroh` / `p2p_legacy` | Web4 transport (SHIP-4) and legacy WebSocket bridge (SHIP-5) |
| `delegate` | round computation, slot scheduling, block forging with pluggable delegate keys |
| `mempool` | validation, sender ordering, relay, count and byte budgets |
| `api` | Axum REST server, operator metrics page (SHIP-8) |
| `crypto` | Schnorr / ECDSA, BIP-39, ML-DSA-44 (SHIP-18/20), serializers (SHIP-11/19) |

### Operational profile

- Single static binary per platform; configuration in `node.yaml`; database directory portable between versions with
  identical milestones.
- Resource targets: < 300 MB RSS while synced, < 2 GB disk for full mainnet state, start-up < 2 s.
- Deterministic: the same inputs produce the same state on every node; no background mutation of consensus data.
- `minCoreVersion` milestone field lets operators see which delegates fall behind the rules of the current height (SHIP-8).

### Delegate forging

Forging is pluggable: passphrases or pre-derived keys are configured per node; the same binary acts as a relay when no
delegate is configured. Slot timing, round ordering and block payload limits follow the milestone in force.

## Rationale

Rust was chosen over Go/C++ for memory safety without garbage-collection pauses inside the slot budget, and for a mature
ecosystem covering QUIC (iroh), embedded storage (sled) and post-quantum cryptography. Sled replaces PostgreSQL because the
state model is a set of key prefixes with point lookups and range scans, not relational queries.

## Benefits

30–100× lower resource use, seconds instead of minutes to start, fork rollback without manual intervention, and a code base
in which consensus rules are explicit functions with unit-tested vectors.

## Backwards Compatibility

Full: legacy and Rust nodes co-exist in one network. Features that legacy nodes cannot validate are gated behind milestones
that are scheduled only when all active delegates run `sth-core` (see SHIP-1 workflow).

## Reference Implementation

`sth-core-rust/` - the whole crate; `tests/` (25 suites) including `crypto_vectors`, `genesis`, `sync`, `storage`, `api`.
