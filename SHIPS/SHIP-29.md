```
  SHIP: 29
  Title: Confidential Transfers - Shielded Pool with Post-Quantum Proofs
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-25
  Last Update: 2026-08-21
```

## Abstract

An opt-in **shielded pool** for STH (later tokens): users move coins into the pool, transfer them inside it with hidden
sender, recipient and amount, and withdraw to a transparent address. Privacy comes from note commitments and nullifiers
proven with a **hash-based, transparent STARK** (no trusted setup, post-quantum sound - consistent with Quantum Shield),
while the transparent ledger, fees and DPoS stay unchanged. Amounts entering and leaving the pool are public, so the
supply is always auditable.

## Motivation

Public balances expose payroll, business flows and personal wealth. Ring signatures (Monero-style) leak over time and
scale poorly; pairing-based SNARKs need trusted setups and are broken by quantum computers - unacceptable for a chain
that is adding ML-DSA. A STARK-based shielded pool offers the strongest privacy per byte that remains quantum-sound.

## Specification (design)

### Notes and commitments

A note = `(value, ownerPk, rho, r)`; commitment `cm = Poseidon2(value ‖ ownerPk ‖ rho ‖ r)`. All commitments live in an
append-only Merkle tree (depth 32, Poseidon2, root stored per block in state). Spending reveals a **nullifier**
`nf = Poseidon2(sk ‖ rho)`; nullifiers are stored in `nf:` and may never repeat.

### Transactions (`typeGroup 5`)

| type | name | public inputs | effect |
|---:|---|---|---|
| 0 | Shield | `amount`, `cm` | debits the transparent sender by `amount + fee`, appends `cm` |
| 1 | PrivateTransfer | `root`, `nf[2]`, `cm[2]`, `fee`, `proof` | proves: notes exist under `root`, owner knows `sk`, `Σ in = Σ out + fee`, values in range; appends outputs, records nullifiers |
| 2 | Unshield | `root`, `nf`, `amount`, `recipient`, `proof` | credits the transparent recipient |

Proof system: STARK over a small prime field (Goldilocks / Baby Bear) with FRI, Poseidon2 arithmetisation, proof size
~40–80 KB, verification ~2–5 ms in Rust; **no trusted setup**, security from collision-resistant hashing only (quantum-sound).
Fees are paid publicly (`fee` is a public input) so mempool ordering and burn policies work unchanged.

### Anonymity set and network privacy

Every pool note is indistinguishable; 2-in/2-out fixed arity hides transaction shape. Node-level privacy is handled by
Iroh (encrypted, relayed connections, [SHIP-4.md](SHIP-4.md)); wallets should submit shielded transactions through a random relay peer.

### Keys

Spending key from the wallet's second (PQ) passphrase domain `"sth-shield-v1"` so a Quantum Shield user has one backup
for both; viewing keys allow selective disclosure (audits, tax) without spending power.

### Limits and parameters

Milestone `shield: { active, maxProofBytes: 131072, feePerProofByte }`; block payload cap raised accordingly. Pool
balance = `Σ Shield − Σ Unshield` is a public counter - the pool cannot inflate STH.

## Rationale

Compared with alternatives: ring signatures/CT (Monero) - linkability over time, no quantum soundness for Pedersen
commitments; Groth16/PLONK pools (Zcash Sapling-style) - trusted setup or pairing assumptions broken by quantum computers;
mixers - weak anonymity set. A STARK pool is larger per transaction but future-proof and setup-free; proof size falls with
recursion/aggregation (batch several transfers per block-level proof - later SHIP).

## Security Considerations

Circuit bugs are catastrophic (silent inflation); the public pool counter and per-epoch supply audits bound the damage
window. Requires a formal review before Accepted.

## Backwards Compatibility

New typeGroup; [Rust-only network](https://github.com/smartholdem/sth-core-rust).

## Reference Implementation

Not started; candidate libraries: Plonky3 / Winterfell (STARK), Poseidon2 (hash).
