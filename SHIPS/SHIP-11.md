```
  SHIP: 11
  Title: Transaction Wire Format v2 and Schnorr Signatures
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Final
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-01-05
  Last Update: 2026-06-19
```

## Abstract

This SHIP documents, as a frozen standard, the byte layout of version-2 transactions and the signature schemes in force on
mainnet: Schnorr (BIP-340-style, 64 bytes) for transaction senders and second signatures, and Schnorr block signatures.
Historically these rules were activated by the milestone flags `SHIP-11` (v2 format) and `SHIP-11` (Schnorr blocks); the keys
stay in `milestones.json` for wallet-library compatibility, the specification lives here. Version 3 (SHIP-19) extends this
layout without changing any byte of the common header.

## Specification

### Common header (v2)

```
0xff                       marker
u8   version = 2
u8   network (pubKeyHash)
u32  typeGroup           LE
u16  type                LE
u64  nonce               LE
33 B senderPublicKey     compressed secp256k1
u64  fee                 LE
u8   vendorField length || vendorField (≤ 255 bytes UTF-8)
```

### Type payloads (typeGroup 1)

|                                  type | payload                                        |
|--------------------------------------:|------------------------------------------------|
|                            0 transfer | `u64 amount ‖ u32 expiration ‖ 21 B recipient` |
|                    1 second signature | `33 B publicKey`                               |
|               2 delegate registration | `u8 len ‖ username`                            |
|                                3 vote | `u8 count ‖ (0x01/0x00 ‖ 33 B delegate)*`      |
|                      4 multisignature | `u8 min ‖ u8 count ‖ 33 B *`                   |
|                            5 SMARTNET | `hash bytes`                                   |
|                        6 multipayment | `u16 count ‖ (u64 amount ‖ 21 B recipient)*`   |
|                7 delegate resignation | -                                              |
| 8 / 9 / 10 HTLC lock / claim / refund | as legacy core                                 |

typeGroup 2 (sObjects) is defined in SHIP-13; typeGroup 3 (tokens) in SHIP-14.

### Signatures

```
signature        64 B Schnorr over sha256(bytes without signatures)
secondSignature  64 B Schnorr over sha256(bytes with signature, without second signature)   (optional)
signatures       65 B each (index ‖ signature) for multisignature wallets                   (optional)
```

Transaction id = `sha256(full bytes)`. Legacy ECDSA (DER) signatures remain valid for historical blocks only.

### Blocks

Header fields serialised LE as in the legacy core; `blockSignature` = Schnorr over `sha256(header)`; block id =
`sha256(header ‖ signature)`; `payloadHash = sha256(concat(transaction ids))`.

## Rationale

Freezing the format in a SHIP gives wallets and explorers a single reference independent of legacy source code; the
serializer in `sth-core` is verified against published vectors and against every historical block.

## Reference Implementation

`src/crypto/tx_serializer.rs`, `src/crypto/tx_deserializer.rs`, `src/crypto/block_serializer.rs`, `src/crypto/schnorr.rs`;
tests `tests/crypto_vectors.rs`, `tests/deserializer.rs`, `tests/genesis.rs`.
