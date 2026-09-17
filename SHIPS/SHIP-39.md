```
  SHIP: 39
  Title: Sub-Second Slots and Pipelined Forging
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-17
  Last Update: 2026-09-11
```

## Abstract

A staged reduction of the block time from 8s to 4s, 2s and finally **1s**, made possible by validation
([SHIP-2.md](SHIP-2.md) / [SHIP-7.md](SHIP-7.md)), sub-100ms gossip ([SHIP-4.md](SHIP-4.md)), compact blocks and parallel apply ([SHIP-36.md](SHIP-36.md)) and explicit finality ([SHIP-35.md](SHIP-35.md)).
Forging becomes a pipeline - the next block's transactions are verified while the current one propagates - and delegates are
required to meet latency and clock criteria enforced by the protocol.

## Motivation

8-second slots date from a network of Node.js peers on WebSockets. User-perceived speed (payments confirmed in ~1s,
market orders settled instantly) and throughput per second both scale with slot count; the measured budget shows a 1-second
slot is feasible when the network is [sth-coreRust-only](https://github.com/smartholdem/sth-core-pq) (`docs/THROUGHPUT-LIMITS.md`, `docs/MULTIPAY-1024.md`).

## Specification

### Stages (each a milestone `{ blocktime, maxTransactions, maxPayload }`)

| stage | blocktime | max tx / block | requirements |
|---|---:|---:|---|
| S1 | 4 s | 1 000 | all delegates on `sth-core` ≥ [SHIP-36.md](SHIP-36.md); compact blocks on |
| S2 | 2 s | 1 500 | SHIP-35 finality active; delegate latency policy below |
| S3 | 1 s | 2 000 | SHIP-32 lanes ≥ 4 or apply ≤ 150 ms measured; NTP discipline |

Budget rule: `apply + verify + propagate ≤ 50%` of the slot on the reference machine (8 cores, 100 Mbit/s).

### Pipelining

The forger of slot `n+1` pre-validates mempool transactions against the state *after* block `n` as soon as block `n` is
received (optimistic), and forges immediately at slot start; `previousBlock` binding stays sequential. Nodes verify block
`n+1` signatures while applying block `n` (already parallel). Round schedule unchanged (21 delegates, shuffled per round).

### Delegate latency policy

`GetStatus` gains `clockOffsetMs` (NTP-measured); the Delegate Dashboard ([SHIP-8.md](SHIP-8.md)) shows offset and block arrival delay per
delegate; a delegate missing > 10% of slots over a round is flagged. Slot tolerance for late blocks tightens per stage
(`blockTimeTolerance`: 2s -> 500ms -> 200ms).

### Reward and fees

Block reward per block is divided by the slot-count factor so daily issuance is unchanged (`reward` milestone adjusted with
the same height); fees unchanged.

## Rationale

Incremental stages with measurable gates avoid the failure mode of "fast chains" that skip slots under load; every stage is
individually reversible by milestone ([SHIP-23.md](SHIP-23.md) governance may vote on it).

## Backwards Compatibility

[sth-core-pq-only](https://github.com/smartholdem/sth-core-pq)  network; each stage is a milestone with `minCoreVersion`.

## Reference Implementation

Not started; `delegate/forger.rs` (pipeline), `config` (tolerances), `bench_block`.
