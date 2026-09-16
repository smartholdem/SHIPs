```
  SHIP: 22
  Title: Quantum Shield Stage C - Hybrid Block Signatures
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-04
  Last Update: 2026-08-08
```

## Abstract

Protect the consensus itself: blocks of version 1 carry **two** signatures of the forging delegate - the existing Schnorr
signature over the header and an ML-DSA-44 signature over `sha256(header ‖ blockSignature)`. Both must verify. The
delegate's PQ key is the PQ key of its wallet (registered with [SHIP-20.md](SHIP-20.md)), activation is the milestone `pq.blocks` with a
grace period during which v0 blocks are still accepted.

## Motivation

Stage B protects user balances, but a quantum adversary who recovers a delegate's secp256k1 key from its public key
(present in every block) could forge blocks in the delegate's slots. Hybrid signatures close this while keeping
`generatorPublicKey`, delegate addresses and votes unchanged, and hedge against an unknown weakness in either scheme.

## Specification

### Block v1

| field | v0 | v1 |
|---|---|---|
| `version` | 0 | 1 |
| header | unchanged | unchanged |
| `blockSignature` | Schnorr 64 B over `sha256(header)` | unchanged |
| `pqSignature` | - | `u8 alg (1) ‖ u16 len (2420) ‖ ML-DSA-44.Sign(sk, M_B, ctx = "sth-pq-block-v1")` |

`M_B = sha256(header ‖ blockSignature)`; block id = `sha256(all bytes incl. both signatures)`. Size +2 423 B per block
(≈ 26 MB/day at 8-second slots).

### Delegate key

The PQ key of the wallet owning `generatorPublicKey` (`wallet.pq_key`), registered/rotated with `pq-register`. A key
change takes effect from the **next** block (state before the block is used).

### Milestone and transition

`{ "height": H_C, "pq": { "blocks": true, "blocksGrace": 43200 } }` - from `H_C` v1 blocks are accepted and expected;
v0 blocks are accepted until `H_C + blocksGrace`, then rejected (`BlockVersionError`). A delegate without a PQ key after the
grace period misses its slots (`BlockPqKeyMissingError`); the Delegate Dashboard (SHIP-8) shows `PQ key: missing` and a
countdown beforehand.

### Verification order

1. version allowed at this height; 2. Schnorr signature (as today, in the parallel pool); 3. `pqSignature` algorithm/length,
`verify(pq_key(generator), M_B)`; 4. undo: `pq_key` lives in wallet state - no special rollback logic.

### Forger

`delegate.pq_secrets` / `STH_DELEGATE_SECOND_PASSPHRASE` in `node.yaml`; `sth-core info` prints `pq key: registered / missing`.

## Rationale

Signing the classical signature (not just the header) binds the two schemes; reusing the wallet PQ key avoids a new
transaction type and lets delegates prepare during Stage B.

## Open questions

Hybrid forever vs. PQ-only Stage D (proposal: hybrid ≥ 2 years); FN-DSA-512 for smaller block signatures (`pq.blockAlgorithms`);
quantum-safe addresses (separate SHIP).

## Backwards Compatibility

Rust-only network (legacy P2P cannot parse v1 blocks); after SHIP-20 activation and legacy node retirement.

## Reference Implementation

Not started; design in `docs/PLAN-QUANTUM-SHIELD.md` §C.1–C.6.
