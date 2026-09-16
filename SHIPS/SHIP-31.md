```
  SHIP: 31
  Title: State and History Pruning
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-29
  Last Update: 2026-06-19
```

## Abstract

Node modes that keep full validation guarantees while discarding data not needed for consensus: **pruned** nodes keep the
current state, the last `N` blocks with undo records and a chain of **state commitments** ([SHIP-38.md](SHIP-38.md)); **archive** nodes keep
everything. Pruned nodes sync from a state snapshot whose root is signed into the chain by delegates, then validate blocks
forward as usual.

## Motivation

Full history grows without bound (2 GB today, several GB per year after tokens and PQ). Delegates and relays need only the
state to validate new blocks; explorers need history. Separating the roles lowers the cost of running a validating node -
more nodes, more decentralisation.

## Specification

- `node.yaml -> storage.mode: archive | pruned`, `storage.keep_blocks: 100000` (≥ 2 * fork-rollback depth).
- Pruned node: after finality depth `F` ([SHIP-35.md](SHIP-35.md) gives explicit finality; until then `F = 10 000`) delete `b:`/`t:`/`tl:`/`wt:`
  entries older than `keep_blocks`, keep wallet state and indexes, keep per-epoch **state root** `sr:<height>`.
- **State root**: Merkle (Poseidon2 or SHA-256, binary) over sorted `w:` entries + token/sObject/lock tables, computed
  every 1000 blocks by all nodes and included by the forger in the block header extension `stateRoot` (block v1 field,
  together with [SHIP-22.md](SHIP-22.md)) - a mismatch is a consensus failure, so pruned nodes verify that the snapshot they start from is
  the network's state.
- **State snapshot**: `sth-core snapshot state --height H` produces `state-H.tgz` (all state entries); `import --state`
  verifies its root against `sr:<H>` in a header obtained from peers, then follows.
- API on pruned nodes: `/api/blocks/:id`, `/api/transactions/:id` return `410 Gone` with `archiveHint` (list of archive
  peers announced over gossip, [SHIP-4.md](SHIP-4.md)) for pruned heights; wallet, token, sObject and market endpoints are complete.
- History serving: archive nodes announce `archive: true` in `GetStatus`; sync prefers them for deep ranges.

## Rationale

State roots in headers are the one consensus change and are needed anyway for light clients ([SHIP-38.md](SHIP-38.md)); everything else is
local policy, so nodes can adopt pruning independently.

## Backwards Compatibility

Header extension requires block v1 (Rust-only network, [SHIP-22.md](SHIP-22.md)). Pruning without roots (trusting a snapshot) is possible
earlier as an operator choice but is not recommended for delegates.

## Reference Implementation

Not started.
