```
  SHIP: 21
  Title: Wallet Quantum Shield - Client Requirements
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Interface
  Created: 2026-04-27
  Last Update: 2026-06-19
```

## Abstract

Requirements for wallets (web, desktop, mobile, hardware bridges) implementing Quantum Shield ([SHIP-18.md](SHIP-18.md), [SHIP-19.md](SHIP-19.md), [SHIP-20.md](SHIP-20.md)): key
derivation, user flows, signing, error handling and test conformance, so that every client produces the same bytes and
gives users the same guarantees.

## Motivation

A second lock is only as safe as the client that manages it. Wallets must derive keys identically (or funds become
unspendable), must never register a key the user cannot back up, and must handle the network's state transitions (grace
window, PQ-locked status) without surprising the user.

## Specification

### Derivation and storage

- ML-DSA-44 key from the **second passphrase** exactly as [SHIP-18.md](SHIP-18.md) (`sha256("sth-pq-v1" ‖ passphrase)` -> KeyGen). The
  passphrase is shown once, must be confirmed by re-entry, and is stored encrypted like the first one; the public key may be
  cached.
- Conformance: the wallet must reproduce `tests/vectors/pq_v3.json -> vectors` (seed, pk, sig) and all `transactions`
  (`fullHex`, `id`) bit-for-bit before shipping.

### Flows

1. **Status** - read `GET /api/wallets/:addr -> quantumShield` and `GET /api/node/configuration -> pq`; show one of
   *not protected / committed (Stage A) / ACTIVE since block N*; show `pq.active` for the network.
2. **Commit (Stage A)** - button available any time; builds the self-transfer with `sthpq1:` memo; explains the grace
   window.
3. **Activate (Stage B)** - enabled when `pq.active`; requires the second passphrase (new) and, if the wallet has a
   legacy second signature or a PQ key, the old one; shows the fee incl. surcharge; sends the v3 registration; polls until
   `quantumShield.active`.
4. **Send** - if `quantumShield.active`, every transaction is built as v3 with one ML-DSA block and `fee += surcharge`
   computed from `pq.feePerByte`; if `secondPublicKey` is set, legacy v2 second signature; otherwise v2.
5. **Rotate** - Activate flow with the old PQ passphrase as proof.

### Errors to handle

`ERR_PQ_NOT_ACTIVE` (show activation height), `ERR_PQ_SECOND_SIGNATURE_REQUIRED` (wallet is locked - ask for the
second passphrase), `ERR_PQ_SECOND_SIGNATURE_INVALID` (wrong passphrase), `ERR_PQ_LEGACY_PROOF_REQUIRED`,
`ERR_PQ_COMMITMENT_MISMATCH` (explain the window and the committed key), `ERR_PQ_FEE`.

### UX safeguards

- Irreversibility warning with explicit acknowledgement before registration.
- Never derive or transmit the second passphrase to any server; signing happens locally.
- Display the PQ public key fingerprint (first 16 hex of `sha256(pk)`) so users can compare across devices.

## Rationale

Codifying client behaviour prevents divergent derivations (the single most dangerous failure) and makes the network-wide
switch predictable for support teams and exchanges.

## Reference Implementation

Node side: [SHIP-18.md](SHIP-18.md),[SHIP-19.md](SHIP-19.md),[SHIP-20.md](SHIP-20.md). Client reference: `sth-cli` (`src/cli.rs` sign_with_second, `src/bin/sth-cli.rs`).
