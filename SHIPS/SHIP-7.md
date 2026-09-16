```
  SHIP: 7
  Title: Parallel Signature Verification and Fast Vote Index
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2025-11-10
  Last Update: 2026-06-19
```

## Abstract

Two performance rules of `sth-core` that do not change consensus but define what a block costs to validate: all
signature checks of a block run in parallel across CPU cores, and delegate vote weights are maintained incrementally in an
index instead of being recomputed from every wallet at round boundaries.

## Motivation

Signature verification (~0.24 ms per Schnorr signature) dominates block validation; with 150 transactions per block and
8-second slots the legacy sequential path uses a third of the slot on one core. Delegate ranking in the legacy node scans
all wallets every round (thousands of rows). Both limit block size and slot time ([SHIP-12.md](SHIP-12.md), [SHIP-39.md](SHIP-39.md)).

## Specification

### Parallel verification

- Stateless checks of a block (`verify_block`) - transaction format, fees, payload hash membership, sender signature,
  legacy second signature and ([SHIP-19.md](SHIP-19.md)) v3 blocks - run over the transaction list with a work-stealing pool (rayon).
- Verification is pure: it reads no state and has no ordering requirement, so results are identical to sequential
  verification; errors are collected and reported deterministically (sorted by transaction index).
- Measured: 150 transactions verify in ~9 ms on 4 cores; 5 000 in ~0.3 s.

### Fast vote index

- `dv:<delegate public key> -> u64` holds the current vote weight of each delegate.
- Every balance change of a voting wallet, every vote/unvote and every HTLC settlement adjusts the index by the exact
  delta inside the same atomic block write ([SHIP-3.md](SHIP-3.md)); rollback restores it from the wallet snapshots.
- Round computation reads the top‑N of the index (N = `activeDelegates` = 21) - O(delegates) instead of O(wallets).
- The vote weight of a wallet is its balance (plus locked HTLC balance as in the legacy rules); the index is verified
  against a full recomputation in tests after long random sequences.

## Rationale

Keeping semantics identical while changing only evaluation order and data structures means these rules can ship without a
milestone. They are the prerequisite for larger blocks and shorter slots.

## Reference Implementation

`src/crypto/block_serializer.rs` (`verify_block`), `src/storage.rs` (`adjust_votes`, `delegate_ranking`),
`tests/bench_block.rs`, `tests/delegate.rs`.
