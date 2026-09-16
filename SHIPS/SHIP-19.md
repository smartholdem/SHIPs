```
  SHIP: 19
  Title: Transaction Wire Format v3 - Second-Signature Blocks
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-04-13
  Last Update: 2026-07-21
```

## Abstract

Version 3 of the transaction format keeps the v2 header and every type payload byte-for-byte (SHIP-11) and replaces the
single fixed-size second signature with a list of **algorithm-tagged blocks**, so a transaction can carry a
post-quantum second signature (2 420 bytes) - or several during a key migration - while classical tooling still parses the
body. It also extends the type-1 (second signature registration) payload with an algorithm id and a variable-length key.

## Motivation

The v2 layout hard-codes 64-byte second signatures and 33-byte second public keys. Post-quantum schemes need kilobyte-sized
keys and signatures and a migration path where the old and the new second key sign together once. A generic block list
solves both and leaves room for future algorithms (FN-DSA, hash-based) without another format change.

## Specification

### Header

Identical to v2 except `version = 3`.

### Type 1 payload (second-signature registration)

```
u8  algorithm          1 = ML-DSA-44
u16 pkLen (LE)         1312
pk                     raw public key
```

JSON: `asset.signature = { algorithm, publicKey }`.

### Signature section

```
signature          64 B Schnorr over SIGNING_HASH = sha256(bytes without any signature)
blocks*            u8 algorithm ‖ u16 sigLen (LE) ‖ signature          ascending algorithm order, at most two
[0xff ‖ 65 B*]     multisignature parts (optional, marker 0xff)
```

| algorithm | scheme | sigLen | message |
|---:|---|---:|---|
| 0 | secp256k1 Schnorr (legacy second key) | 64 | `M2` |
| 1 | ML-DSA-44, context `"sth-pq-v1"` | 2420 | `M2` |

`M2 = sha256(bytes with the first signature, without blocks and without multisignatures)` - the same message the v2
second signature signs, so the PQ block *covers* the classical signature.

JSON: `secondSignatures: [{ algorithm, signature }]`; `secondSignature` must be absent in v3.

Transaction id = `sha256(full bytes)` including blocks.

### Sizes

Registration (fresh) 3 861 B · registration with legacy proof 3 928 B · PQ-locked transfer 2 581 B · rotation 6 284 B ·
v3 transfer with alg-0 block 225 B (vectors in `tests/vectors/pq_v3.json → transactions`).

### Validation (stateless, every node)

`ERR_PQ_NOT_ACTIVE` (v3 before milestone `pq.active`), `ERR_PQ_ALGORITHM`, `ERR_PQ_LENGTH` (sizes, ≤ 2 blocks, no
`secondSignature`), `ERR_PQ_ORDER`, `ERR_PQ_FEE` (see [SHIP-20.md](SHIP-20.md)). Wallet-aware rules are in [SHIP-20.md](SHIP-20.md).

## Rationale

Length-prefixed blocks with a 1-byte algorithm tag cost 3 bytes of overhead and make the format self-describing; keeping
the body untouched means block explorers and hardware signers need only a new signature-section parser.

## Backwards Compatibility

v3 is rejected by legacy nodes and by `sth-core` until `pq.active`; v2 stays valid indefinitely for wallets without a PQ key.

## Reference Implementation

`src/crypto/tx_serializer.rs`, `src/crypto/tx_deserializer.rs` (`deserialize_signatures_v3`), `transaction_pq_message`,
`src/models/transaction.rs` (`PqSignatureBlock`, `VERSION_PQ`); tests `tests/pq_v3.rs`; spec `docs/SPEC-PQ-V3.md`.
