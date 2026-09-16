```
  SHIP: 3
  Title: Compact State Database on Sled
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2021-09-22
  Last Update: 2026-06-19
```

## Abstract

This SHIP defines the storage model of `sth-core`: a single embedded Sled key-value tree holding blocks, transactions,
wallet state and every secondary index under short byte prefixes, written atomically per block, with a per-block undo log
that makes fork rollback a local operation. The layout is the canonical description of node state referenced by later
SHIPs (sObjects, tokens, market, Quantum Shield).

## Motivation

The legacy node stores the chain in PostgreSQL: a separate server process, tens of gigabytes, slow cold start and a wallet
state that must be rebuilt in memory. Consensus needs three operations only - point lookup, prefix scan and atomic batch
write - which an embedded log-structured store provides at microsecond latency with a footprint under 2 GB for the whole
mainnet history.

## Specification

### Key prefixes

| Prefix | Key | Value |
|---|---|---|
| `b:` | height (u64 BE) | block with transactions (serde) |
| `bid:` | block id | height |
| `t:` | transaction id | transaction + block coordinates |
| `tl:` / `wt:` | height·seq / address·seq | ordered transaction lists (global, per wallet) |
| `w:` / `wp:` / `wu:` | address / public key / username | `WalletState` (balance, nonce, votes, second key, `attributes`: sObjects, tokens, `pq_key`) |
| `dv:` | delegate public key | vote weight (fast vote index, [SHIP-7.md](SHIP-7.md)) |
| `lk:` | lock id | HTLC lock |
| `rd:` | round | delegate order of the round |
| `en:` / `eo:` / `mk:` | sObject name·type / registration id / registration id | name uniqueness, owner, open sale order ([SHIP-13.md](SHIP-13.md)/[SHIP-16.md](SHIP-16.md)) |
| `tk:` / `tks:` | token id / symbol | token state, symbol index ([SHIP-14.md](SHIP-14.md)) |
| `pqc:` / `pqk:` | address | Quantum Shield commitment / registered key ([SHIP-18.md](SHIP-18.md)/[SHIP-20.md](SHIP-20.md)) |
| `undo:` | height | previous wallet states touched by the block |

Prefixes are historical identifiers; renaming them would force a resync of every node, so they never change.

### Atomicity and undo

Applying a block is one Sled transaction: block, transactions, wallet deltas, indexes and the undo record are committed
together or not at all. The undo record stores the *previous* `WalletState` of every touched wallet (a snapshot, not a
diff); rollback restores those snapshots and recomputes derived indexes (`dv:`, `en:`, `mk:`, `tks:`, `pqc:`, `pqk:`).
Because state lives inside `WalletState` (serde, `camelCase`, `skip_serializing_if` for empty attributes), adding a new
attribute is backwards compatible and automatically covered by rollback.

### Batch application

During synchronisation blocks are validated and applied in batches; wallet-aware rules see the effects of earlier blocks
of the batch through in-memory views (`pending` balances, sObject and token views) so that a batch equals sequential
application.

### Fast import

`snapshot import --fast-import` (SHIP-6) writes blocks without undo records and with signature checks deferred to the
checkpointed verification pass, then re-enables undo for live following.

## Rationale

A single tree with prefixes (instead of one tree per index) keeps atomic commits trivial and lets a backup be a file copy.
Snapshot-style undo costs more bytes than diffs but makes rollback independent of the transaction semantics - new
transaction types need no rollback code.

## Benefits

Cold start in seconds, < 2 GB for mainnet, fork rollback in milliseconds, state inspection with plain prefix scans.

## Backwards Compatibility

Not applicable to the wire protocol. Databases are portable between core versions with identical milestones; new
attributes are additive.

## Reference Implementation

`src/storage.rs` (`Storage`, `WalletState`, `WalletDelta`, `apply_block`, `rollback_to`); tests `tests/storage.rs`,
`tests/sync.rs`, `tests/balance.rs`.
