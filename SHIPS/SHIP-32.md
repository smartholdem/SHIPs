```
  SHIP: 32
  Title: Sharding - Sender-Partitioned Execution Lanes
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-01
  Last Update: 2026-08-30
```

## Abstract

A pragmatic path to horizontal scale that keeps **one chain, one set of 21 delegates and one security domain**: the state
is partitioned into `S` **lanes** by sender address; every block contains `S` independent transaction lists, each applied
in parallel; cross-lane effects (recipient credits, token moves to another lane) are delivered as **receipts** applied in
the next block. Later phases let delegates validate only a subset of lanes with data-availability sampling, without ever
splitting consensus.

## Motivation

Single-threaded state application is the ceiling after signature verification is parallel ([SHIP-7.md](SHIP-7.md)): ~11 000 operations per
second per core. Multi-chain sharding (separate validator sets) weakens security and complicates users' lives. Lane
sharding gives near-linear CPU scaling on one chain and is a prerequisite for the higher tiers of the throughput plan
(`docs/THROUGHPUT-LIMITS.md`).

## Specification

### Lanes

`lane(addr) = first byte of RIPEMD160(pubkey) mod S`, `S ∈ {1, 2, 4, 8, 16}` - a milestone parameter (`lanes`). A transaction
belongs to the lane of its **sender**; its debit (amount, fee, nonce) is applied there.

### Block layout (v1 extension)

`lanes: [ { laneId, txIds[] } ]`, `payloadHash` over the concatenation in lane order; `receiptsRoot` = Merkle root of
outgoing receipts.

### Receipts

A credit to a recipient in another lane produces a receipt `(fromLane, toLane, seq, kind, target, amount/asset)`; receipts of
block `h` are applied at the start of block `h + 1` in the target lane before its transactions. Same-lane credits apply
immediately (as today). Balance available to a recipient is therefore delayed by at most one slot for cross-lane
payments - wallets show "arriving".

### Execution

Each lane is applied by its own thread with its own `pending` view ([SHIP-3.md](SHIP-3.md)); lanes never touch each other's wallets within
a block. Vote weights ([SHIP-7.md](SHIP-7.md)) are aggregated after all lanes finish (deterministic order).

### Phase 2 - lane validation subsets

With `S = 16`, a delegate validates all lanes but may **store** only a subset plus state roots per lane ([SHIP-31.md](SHIP-31.md) / [SHIP-38.md](SHIP-38.md)); erasure-
coded lane data with sampling lets relays verify availability without downloading every lane. Consensus remains one
DPoS round; no lane has its own validator set.

### Milestone

`{ "height": H, "lanes": 4, "blockVersion": 1 }`; `S` may only increase (re-partitioning by doubling keeps `lane` a prefix
function, so existing state moves deterministically).

## Rationale

Sender-partitioning makes every transaction's debit side local, which is where all conflicts (nonce, balance) live; the
receipt delay is the only user-visible change and mirrors how banks post credits.

## Backwards Compatibility

Block v1, [Rust-only network](https://github.com/smartholdem/sth-core-pq). `S = 1` is exactly today's behaviour.

## Reference Implementation

Not started; depends on [SHIP-36.md](SHIP-36.md) (parallel apply infrastructure).
