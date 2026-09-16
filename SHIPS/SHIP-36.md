```
  SHIP: 36
  Title: Compact Blocks and Parallel State Application
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-10
  Last Update: 2026-09-01
```

## Abstract

Two throughput upgrades that need no consensus change: **compact blocks** - a forged block is announced as a header plus
short transaction ids and peers reconstruct it from their mempools, requesting only what they miss; and **parallel state
application** - transactions of a block are grouped by sender and applied concurrently where they touch disjoint wallets,
with a deterministic merge. Together they move the network from tier T1 (~11 000 operations/s) towards T2 (~45 000) on
8-core delegate hardware.

## Motivation

At 150 transactions per block, propagation and apply time are small; at 5 000+ (larger blocks, [SHIP-12.md](SHIP-12.md) multipayments,
[SHIP-39.md](SHIP-39.md) shorter slots) a full block is hundreds of kilobytes gossiped to every peer that already holds 95% of its
transactions, and single-threaded apply becomes the slot bottleneck.

## Specification

### Compact blocks (gossip topic `blocks`, message `CompactBlock`)

`{ header, shortIds: [u48], prefilled: [(index, tx)] }`; `shortId = SipHash-2-4(txId, key = sha256(header))[0..6]`.
Receiver looks up short ids in its mempool; missing ones are fetched from the sender with `GetBlockTxn { blockId, indexes }`
(RPC, [SHIP-4.md](SHIP-4.md)); after reconstruction `payloadHash` is verified - a mismatch falls back to `GetBlocks`. Forgers include as
`prefilled` any transaction they received less than 1 second before forging. Expected size: 6B per transaction plus header
(~30 KB for 5 000 transactions instead of ~800 KB).

### Parallel apply

1. Verify signatures in parallel ([SHIP-7.md](SHIP-7.md)).
2. Build the **conflict graph**: two transactions conflict if they touch a common wallet (sender, recipients, delegate for
   votes, token owner/holders, sObject owner, lock parties). Recipients-only overlaps are commutative credits and are handled
   by per-wallet atomic accumulators, not conflicts.
3. Connected components are applied by a thread pool, each on its own pending view ([SHIP-3.md](SHIP-3.md)); nonces and balances are checked
   inside the component in block order.
4. Deterministic merge in transaction order into one Sled batch; vote index ([SHIP-7.md](SHIP-7.md)) updated from the merged deltas.
   Result is byte-identical to sequential application (property-tested).

### Metrics

`bench_block` reports apply throughput per core; the metrics page ([SHIP-8.md](SHIP-8.md)) shows `apply ms`, `compact hit %`.

## Rationale

Both techniques are proven in other networks (BIP-152 compact blocks; optimistic/parallel execution) and are purely local
optimisations: any node may adopt them independently, and they are the groundwork for lane sharding ([SHIP-32.md](SHIP-32.md)).

## Backwards Compatibility

Full; legacy peers keep receiving full blocks over the bridge ([SHIP-5.md](SHIP-5.md)).

## Reference Implementation

Not started; `src/p2p_iroh/gossip.rs` (`CompactBlock`), `src/sync.rs` (`apply_parallel`), `tests/bench_block.rs`.
