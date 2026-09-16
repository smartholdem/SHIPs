```
  SHIP: 20
  Title: Quantum Shield Stage B - ML-DSA-44 Second Signature
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-04-20
  Last Update: 2026-07-28
```

## Abstract

Stage B makes wallets quantum-resistant **without changing addresses or first keys**: a wallet registers an ML-DSA-44
public key ([SHIP-19.md](SHIP-19.md) v3, type 1) and from then on every transaction must carry a valid ML-DSA-44 block - a second lock a
quantum adversary holding the secp256k1 key cannot open. Legacy second signatures migrate to PQ keys with a proof of the
old key; keys can be rotated; a Stage A commitment ([SHIP-18.md](SHIP-18.md)) is binding during a grace window. Everything is gated by the
milestone `pq`.

## Motivation

Replacing secp256k1 for all users at once is impossible (addresses, exchanges, hardware wallets). A second, opt-in lock
protects balances immediately for those who want it, keeps the first key for compatibility, and gives the ecosystem years
to move the primary scheme ([SHIP-22.md](SHIP-22.md) and later).

## Specification

### Milestone

```json
{ "height": H, "pq": { "active": true, "feePerByte": 10000, "commitmentGrace": 86400 } }
```
`activation = H`; `sth-core init newnet --pq-at H` for test networks.

### Wallet state

`WalletState.pq_key = { algorithm, publicKey, since }`; registering clears `secondPublicKey`. A wallet with `pq_key` is
**PQ-locked**. Index `pqk:` for metrics; API `quantumShield { active, algorithm, publicKey, since }`.

### Rules (`check_pq`, evaluated before the legacy second-signature rule)

| wallet state | v2 transaction | v3 transaction |
|---|---|---|
| no second key | valid | blocks must be empty (`UnexpectedSecondSignatureError`) |
| legacy second key | legacy second signature required | exactly one block alg 0 signed by the legacy key |
| PQ-locked | **rejected** `ERR_PQ_SECOND_SIGNATURE_REQUIRED` | exactly one block of the registered algorithm, valid over `M2` (`ERR_PQ_SECOND_SIGNATURE_INVALID`) |

**Registration** (`typeGroup 1`, `type 1`, v3, `asset.signature.algorithm = 1`, 1 312-byte key):

- no second key -> blocks `[1 by new key]`;
- legacy second key -> `[0 by legacy key, 1 by new key]` (`ERR_PQ_LEGACY_PROOF_REQUIRED` otherwise);
- PQ-locked (rotation) -> `[old alg by old key, 1 by new key]`.
- commitment present and `height < activation + commitmentGrace` -> `sha256(newKey)` must equal the commitment
  (`ERR_PQ_COMMITMENT_MISMATCH`); after the window any key.

Effects apply within a block and inside the mempool (a pending registration already locks the sender).

### Fee

`surcharge = feePerByte * Σ(3 + sigLen)` over the blocks - 0.2423 STH per ML-DSA block, 0.0067 STH per alg-0 block.
Core types: `fee ≥ staticFee(type) + surcharge` (`ERR_PQ_FEE`); sObject and token types: `fee = exact + surcharge`.

### Tooling

`sth-cli tx pq-register [--old-second-passphrase]`; `--second-passphrase` / `STH_SECOND_PASSPHRASE` makes every `sth-cli tx`
produce v3 automatically for PQ-locked wallets; `sth-cli wallet` shows `quantum shield ACTIVE · ML-DSA-44 since block N`.
Rollout: `docs/MAINNET-ROLLOUT-PQ.md`.

## Rationale

Requiring **exactly one** block for locked wallets (not "at least one") keeps verification cost bounded and rules
unambiguous; letting `feePerByte` and the grace window live in the milestone means economics can follow real usage.

## Security Considerations

The first key still authorises the transaction; PQ protects against theft of funds, not against a quantum adversary
*preventing* a user from spending (they would need the PQ key too). Loss of the second passphrase is unrecoverable - wallets
must warn before registration ([SHIP-21.md](SHIP-21.md)).

## Backwards Compatibility

Milestone-gated; requires all 21 active delegates on `sth-core` (legacy nodes cannot parse v3).

## Reference Implementation

`src/rules.rs` (`check_pq_format`, `check_pq`, `PqWalletView`), `src/sync.rs`, `src/mempool.rs`, `src/storage.rs`,
`src/api/node.rs`, `src/cli.rs`; tests `tests/pq_v3.rs`; docs `docs/SPEC-PQ-V3.md`, `docs/RELEASE-0.19.md`.
