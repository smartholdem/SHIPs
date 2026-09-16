```
  SHIP: 6
  Title: Snapshots: Dump Import, Fast Import and Download
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2025-10-27
  Last Update: 2026-06-19
```

## Abstract

This SHIP specifies how a node bootstraps from a snapshot instead of replaying every block from peers: the archive format
(compatible with dumps produced by the legacy node), the download source, the fast-import mode and the verification pass that
guarantees the imported chain equals the one validated block by block.

## Motivation

Replaying ~10 million blocks from peers takes hours and depends on peer bandwidth. Operators need a way to stand up a relay
or a delegate node in minutes while keeping the guarantee that no invalid history can be injected through a snapshot.

## Specification

### Archive format

`<start>-<end>.tgz` containing `meta.json` and gzip streams of msgpack records:
- `blocks` - `[id, version, timestamp, previousBlock, height, numberOfTransactions, totalAmount, totalFee, reward,
  payloadLength, payloadHash, generatorPublicKey, blockSignature]`;
- `transactions` - `[id, blockId, blockHeight, sequence, timestamp, serialized]` where `serialized` is the SHIP-11 wire
  bytes (so transactions are re-parsed by the node's own deserializer, never trusted as JSON);
- `rounds` - delegate ranking per round (ignored by relays; recomputed from votes).

### Commands

```
sth-core snapshot download --out ./snapshots        # newest archive from snapshots.smartholdem.io (or sync.bootstrap_snapshot)
sth-core snapshot info <dir>                        # meta.json summary
sth-core snapshot import <archive> [--fast-import]  # into the node database
```

### Fast import

With `--fast-import` blocks are written without undo records and signature verification is deferred: after every
checkpoint (10 000 blocks) the node verifies `payloadHash`, block signatures and transaction signatures of the checkpointed
range in parallel (SHIP-7) and aborts on the first mismatch. State transitions are always applied through the normal rules;
only the undo log is skipped. Undo is re-enabled when the import ends, before the node starts following peers.

### Safety

The snapshot is *untrusted input*: block ids, previous-block links, signatures, fees and balances are all re-validated. A
node started from a snapshot ends in a state byte-identical to a node synced from genesis (verified by the `snapshot`
test suite on a real dump range).

## Rationale

Reusing the legacy dump format lets existing operators reuse their archives and the public snapshot server; verifying in
checkpointed batches keeps import CPU-bound instead of I/O-bound.

## Benefits

Full mainnet node in minutes; no trust in the snapshot provider; the same archives serve legacy and Rust nodes.

## Reference Implementation

`src/snapshot.rs`; `tests/snapshot.rs`.
