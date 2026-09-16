```
  SHIP: 18
  Title: Quantum Shield Stage A - Post-Quantum Key Commitments
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-04-06
  Last Update: 2026-07-19
```

## Abstract

Stage A of Quantum Shield lets any wallet **commit today** to the post-quantum key it will register later (SHIP-20) by
publishing `sha256(publicKey)` in the `vendorField` of an ordinary self-transfer. Nodes index commitments; when Stage B
activates, a committed wallet may register only the committed key during a grace window, which defeats an attacker who
already holds the wallet's secp256k1 key.

## Motivation

The dangerous moment of any post-quantum migration is the switch itself: whoever registers a PQ key first owns the wallet.
If a classical key has leaked (or is broken by a quantum adversary in the future), the legitimate owner needs a way to
pre-claim the second lock with no new transaction type and no delegate upgrade - and to do so before the adversary knows
the rules are coming.

## Specification

### Key derivation (deterministic, from a second passphrase)

```
xi        = SHA256( UTF8("sth-pq-v1") ‖ UTF8(secondPassphrase) )
(pk, sk)  = ML-DSA-44.KeyGen_internal(xi)               # FIPS 204, pk = 1 312 bytes
```

### Commitment

`vendorField = "sthpq1:" ‖ alg (2 hex, 01) ‖ ":" ‖ hex(SHA256(pk))` - 77 characters, in a transfer whose sender equals the
recipient (amount ≥ 1 smartoshi, normal fee). Any transaction type with a vendorField is accepted; the self-transfer form
is recommended so the commitment costs only the fee.

### Node behaviour

- Every applied block scans vendorFields with the `sthpq1:` prefix; the latest valid commitment per sender is stored in
  `WalletState.pq_commitment { algorithm, commitment, height }` (undo-safe) and indexed in `pqc:` for metrics.
- `GET /api/wallets/:id -> quantumShield { committed, algorithm, commitment, height }`;
  `GET /api/node/configuration -> quantumShield { stage, algorithms, commitmentPrefix }`; metrics page counts committed wallets.
- No consensus rule changes: legacy nodes see an ordinary transfer with a memo. The commitment becomes binding only through
  [SHIP-20.md](SHIP-20.md)'s grace rule (`ERR_PQ_COMMITMENT_MISMATCH`).

### Tooling

`sth-cli pq-commitment --second-passphrase P` prints the commitment and the ready `sth-cli tx transfer ... --memo` command.
Vectors: `tests/vectors/pq_v3.json` (`seed`, `vectors`).

## Rationale

ML-DSA-44 (NIST FIPS 204) was selected for standardisation status, 2.4 KB signatures and fast verification (~0.1 ms);
hash-committing the key (rather than publishing it) keeps Stage A at 77 bytes per wallet and reveals nothing before it is
needed. The domain string `sth-pq-v1` prevents cross-protocol reuse of the derived key.

## Backwards Compatibility

Full - a vendorField convention only.

## Reference Implementation

`src/crypto/pq.rs` (`PqKeyPair`, `commitment`, `parse_commitment`), `src/storage.rs` (`pq_commitment`),
`src/api/render.rs`; tests `tests/pq.rs`.
