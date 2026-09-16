```
  SHIP: 40
  Title: Post-Quantum E2EE Key Registry and Encrypted On-Chain Memos (ML-KEM)
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-19
  Last Update: 2026-09-14
  Requires: 20, 37
```

## Abstract

The **Netfory Messenger** (built into the SmartNet protocols) already uses the SmartHoldem chain as its key directory: a user
publishes once an **X25519** key and an **iroh NodeId**, and peers deliver end-to-end encrypted messages directly over Iroh
without servers. This SHIP (1) extends that registry to **post-quantum** keys - **ML-KEM-768** (FIPS 203) alongside X25519 -
so messenger sessions become quantum-safe with no change to the transport; and (2) reuses the same keys for **encrypted
on-chain memos**: short messages attached to transactions (or sent alone for a fee) that stay in the chain forever, readable
only by sender and recipient. Everyday chat stays off-chain (free, fast); the chain carries what must be permanent.

## Motivation

Messenger keys published today (X25519) are vulnerable to "harvest now, decrypt later": recorded traffic becomes readable
once a quantum computer exists. The registry is on chain, so upgrading it is a protocol matter, not an app update.
Separately, `vendorField` is public: invoice references, payment notes and short statements that people *want* anchored
forever (proof of a message at a height) leak content to everyone. Both problems share one solution - a PQ key registry
and a hybrid encryption format that clients and the chain agree on.

## Specification

### 1. E2EE key registry (on chain, one transaction per rotation)

`E2eeKeyRegister` (`typeGroup 1`, `type 11`; v3 when the wallet is PQ-locked, SHIP-20):

```
u8  count
( u8 alg ‖ u16 len ‖ key )*     alg 1 = X25519 (32 B)   alg 2 = ML-KEM-768 encapsulation key (1 184 B)
u8  nodeIdLen ‖ iroh NodeId (32 B, optional)
u8  policy                      bit0 paidInbox (see 3), bit1 trustedOnly
u64 inboxPriceSmartoshi         price a stranger pays to open a chat (0 = free)
```

State: `wallet.e2ee = { keys: [{alg, key}], nodeId, policy, inboxPrice, since }` (undo-safe); API
`GET /api/wallets/:addr -> e2ee`; name resolution `u://alice` -> address -> keys ([SHIP-13.md](SHIP-13.md) names). Fee `staticFees.e2eeRegister`
(1 STH) + v3 surcharge. Registration with alg 2 **requires** alg 1 as well (hybrid); the ML-KEM secret is derived from
the second (quantum) passphrase with domain `"sth-e2ee-v1"`, so a Quantum Shield user has one backup for signing and
messaging keys.

### 2. Messenger sessions (off chain, Netfory / Iroh)

Unchanged transport (direct QUIC or relay, gossip topics for groups and channels). Session key derivation becomes hybrid:

```
ss_x   = X25519(eph_x, peer_x)
(ct, ss_k) = ML-KEM-768.Encaps(peer_kem)              # only if the peer published alg 2
k      = HKDF-SHA256(ss_x ‖ ss_k, info = "sth-e2ee-v1" ‖ senderAddr ‖ peerAddr)
```
Messages: AES-256-GCM or XChaCha20-Poly1305 with `k` (double-ratchet on top as today). Groups: the group key is wrapped
to every member with the same hybrid KEM; **ban** = on-chain `Ban` transaction by the owner + group key rotation, as the
messenger does now. Clients show a shield badge when a chat is PQ-hybrid; a chat with a peer that has only alg 1 is
marked "classical". No chain rule reads message content - the chain is the phone book and the payment rail.

### 3. Anti-spam economics (on chain)

`inboxPrice` and `trustedOnly` are consensus data, so a sender sees the price before writing. Opening a chat with a
stranger = a transfer of `inboxPrice` STH to the recipient with `vendorField = "e2ee:open"`; the recipient's client accepts
the session after the transfer confirms. Group entry fees and paid channel posts are ordinary transfers to the owner with
tagged memos; the messenger verifies them by transaction id.

### 4. Encrypted on-chain memo (permanent, private content)

For a private note that must live in the chain forever (payment purpose, contract reference, "proof of message"):

```
(ct, ss_k) = ML-KEM-768.Encaps(recipient_kem)      ss_x = X25519(eph, recipient_x)
k          = HKDF-SHA256(ss_x ‖ ss_k, "sth-memo-v1" ‖ sender ‖ nonce)
memo_ct    = XChaCha20-Poly1305(k, aad = sender ‖ recipient, plaintext)
payload    = 0x02 ‖ eph_x (32) ‖ ct (1088) ‖ memo_ct
```

- **Short form** (recipient has alg 1 only): `payload = 0x01 ‖ eph_x ‖ memo_ct` fits the 255-byte `vendorField` for
  plaintext ≤ 190 bytes - usable today with existing messenger keys.
- **Long / PQ form**: `payload` (≥ 1.2 KB) goes into the new field of an `EncryptedMemo` transaction (`typeGroup 1`,
  `type 12`): `recipient 21 ‖ u16 len ‖ payload (≤ 4 096 B)`, `amount` optional. Fee `staticFees.encryptedMemo` (0.5 STH) +
  `memoFeePerByte` (2 000 smartoshi/B -> +2.4 STH for a full PQ envelope) - the price of permanence; regular
  transfers may also embed the 32-byte hash of a payload stored in Netfory (SHIP-37) for `0.0` extra.
- The chain stores ciphertext only; **only the sender and the recipient** can decrypt (sender keeps `k` or the
  plaintext locally; recipient decapsulates). Sender/recipient *addresses* remain public - full metadata privacy is
  SHIP-29's shielded pool; wallets may use one-time addresses derived from the recipient's X25519 key (stealth) to hide the
  recipient at the cost of the recipient scanning.
- Wallets, 0xID Mail and the messenger render memos inline; the messenger shows on-chain memos in the chat timeline as
  "anchored" messages with block height.

### 5. Errors and limits

`E2eeKeyInvalidError` (sizes / hybrid requirement), `EncryptedMemoTooLargeError`, `EncryptedMemoRecipientHasNoKeyError`
(no alg 1/2 published). Keys may be rotated at any time; old ciphertexts stay decryptable with retained secrets.

## Rationale

The messenger already proves the model (chain = directory + payments, Iroh = transport); adding ML-KEM to the registry is
the minimal change that makes it quantum-safe, and hybrid mode protects against a flaw in either primitive. Keeping chat
off chain and memos on chain lets each channel do what it is good at: speed vs. permanence.

## Security Considerations

No forward secrecy for on-chain memos (they are meant to be permanent); the messenger keeps its ratchet for sessions.
ML-KEM ciphertexts are large - the per-byte fee and 4 KB cap bound block growth. Clients must verify that a published key
belongs to the address (it is signed by the address's transaction) and pin key changes with a warning ("Alice's keys
changed").

## Backwards Compatibility

Alg 1 registrations are the messenger's current behaviour; alg 2 and `type 12` are additive and milestone-gated
(`e2ee: true`). Old clients ignore `0x02` memos as opaque.

## Reference Implementation

Not started in `sth-core`; messenger side lives in the SmartNet client (Netfory). Planned: `crypto/kem.rs` (ML-KEM-768),
`models` (`E2eeKeyRegister`, `EncryptedMemo`), `rules`, `api/render.rs` (`e2ee`), `sth-cli tx e2ee-register | memo`.
