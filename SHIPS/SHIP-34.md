```
  SHIP: 34
  Title: Dead Crypto - Dead Man's Switch and Digital Legacy
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-05
  Last Update: 2026-06-20
```

## Abstract

A native **dead man's switch**: a user defines a heartbeat interval (3, 6, 12 months ...) and a set of actions to be executed
by the network - without intermediaries - if no heartbeat arrives in time: transfer of funds and tokens to heirs, release of
encrypted key material to named recipients, and publication of sealed documents (journalist "insurance files", wills,
credentials). Everything is encrypted until the trigger; the chain only stores commitments and the encrypted payload
pointers, and executes the actions deterministically at the trigger height.

## Motivation

Crypto assets die with their owners; whistle-blowers and journalists rely on trusted third parties to release material if
they disappear; families cannot access accounts. A blockchain with deterministic execution and a censorship-resistant
transport (Iroh, [SHIP-4.md](SHIP-4.md); Netfory, [SHIP-37.md](SHIP-37.md)) is the ideal executor - provided the design leaks nothing before the trigger and
cannot be triggered by anyone else.

## Specification

### Switch (a Safe Contract template, [SHIP-33.md](SHIP-33.md), or a dedicated `typeGroup 7` if SHIP-33 is not yet active)

`SwitchCreate`: `interval u32 (blocks) ‖ grace u32 ‖ actionsCommitment 32 ‖ heirsLen u8 ‖ heirs (21 B each) ‖ ntfryData (encrypted payload pointer)`
Fee 10 STH. The owner's address is the **pulse source**; `deadline = creationHeight + interval`.

### Heartbeat - automatic, from any client

Any transaction signed by the owner resets `deadline = height + interval`. A dedicated `SwitchPulse` (0-amount
self-transfer, fee 0.01 STH) exists for the case when the user sends nothing else. The user is **never expected to remember
the pulse**: it is emitted automatically by the clients of the ecosystem when the owner is demonstrably present -

| client | when a pulse is sent |
|---|---|
| **SmartNet** desktop client (wallet, names, messenger, dApp store) | on unlock with the PIN / seed, at most once per `interval / 4` |
| **0xID Mail** (`@sth` P2P mail) | on unlock, and whenever the user *sends* a signed message (the signature proves presence) |
| **Quantum wallets** (SHIP-21 clients, mobile/desktop) | on unlock, silently, together with the balance refresh |
| `sth-cli` / headless nodes | `sth-cli switch pulse` or cron for power users |

Rules for automatic pulses: the client reads `GET /api/wallets/:addr -> switch { deadline }`; it sends a pulse only if
`deadline − height < interval / 2` (so a normal user pays for ~2 pulses per interval, ≈ 0.02 STH per year for a 6-month
switch); it never sends while the user is merely *viewing* without authenticating. Multi-device is natural: every device
that unlocks the same keys may pulse; the chain keeps the latest deadline.

Why the pulse must be an on-chain transaction: off-chain presence (P2P gossip, mail delivery receipts) is visible only to
some peers; every node must compute the same `deadline`, so only a signed transaction included in a block counts.
With Quantum Shield ([SHIP-20.md](SHIP-20.md)) a pulse requires the PQ block too, so a stolen classical key cannot keep a switch alive **or**
silence it - silence needs both keys or the owner's absence. Clients that hold only a *viewing* session (no signing key
unlocked) cannot pulse - by design.

### Trigger

At `deadline + grace` (grace lets heirs be notified: wallets show "switch pending in N days" to heirs) the node executes
the committed actions **as part of block application** (like fee burning, SHIP-17): no one needs to send a transaction, and
every node computes the same result.

### Sealed material - Netfory decentralised encrypted storage

Documents, key material and messages are **not** stored on chain (only `sha256` / BLAKE3 commitments and pointers). They
live in the **Netfory** decentralised encrypted storage ([SHIP-37.md](SHIP-37.md)): content-addressed blobs replicated by *seeders*
(SmartNet headless seeders earning STH for hosting, plus **cloud seeders** - anti-censorship mirrors in ordinary cloud
storage that hold only ciphertext), retrievable by hash from any node.

- **Encryption before upload**: the client encrypts locally (XChaCha20-Poly1305 with a random content key); the content
  key is split with Shamir `k`-of-`n` to the heirs' public keys (X25519 today, ML-KEM per SHIP-40). Seeders, trackers and
  clouds never see plaintext or keys - the same zero-knowledge model as the SmartNet **Crypto Vault**.
- **Retention for any duration and size**: storage is paid up-front in STH as a *storage deal* recorded in the switch
  (`storageDeal { hash, bytes, untilHeight, paidStH }`); seeders prove possession periodically ([SHIP-37.md](SHIP-37.md) proofs) and are paid
  from the deal per epoch; the deal may be topped up by the owner or any heir. A switch with an expired deal is flagged to
  heirs in their clients long before the trigger.
- **Redundancy**: minimum replication factor 5 across seeders in different regions + 1 cloud mirror; the client re-checks
  availability on every pulse and re-seeds automatically if replicas drop below the threshold.
- **At trigger**: nothing moves - the ciphertext is already everywhere; the trigger record makes the pointer public and
  heirs combine their shares to decrypt. Publication actions (`publish`) may additionally instruct seeders to serve the
  blob under a public name (`sth://<name>.<type>/...`) and to fan it out to the `ntfry-content` topic and mail (`@sth`)
  recipients listed in the sealed action list.

### Actions (committed at creation, revealed at trigger)

The plaintext action list is stored encrypted (`ntfryData` -> Netfory object) under a key split with **threshold secret
sharing** (Shamir, `k`-of-`n`) among the heirs' public keys (X25519 today, ML-KEM per [SHIP-40.md](SHIP-40.md)); `actionsCommitment =
sha256(actions)`. Any `k` heirs can decrypt after the trigger; the network additionally executes the *on-chain* actions
directly from a second, plaintext-but-hashed section that carries only what the chain can do by itself:

| action | effect at trigger |
|---|---|
| `transferSth(to, percent)` / `transferToken(id, to, percent)` | balances move from the owner to heirs |
| `transferSobj(id, to)` | names / tickers / NFTs move |
| `revealKey(to, encryptedBlob)` | blob (≤ 8 KB, encrypted to `to`) is written into the trigger record - readable only by `to` |
| `publish(ntfryData)` | pointer to a sealed Netfory document becomes public; heirs holding the key shares release the plaintext |

### Cancel and update

`SwitchUpdate` (owner, new interval/heirs/commitment) and `SwitchCancel` (owner) any time before the trigger - both are
pulses too.

### Privacy

Before the trigger the chain reveals: that an address has a switch, its interval, heir addresses (optionally
**stealth** heirs via one-time addresses derived by the wallet), a hash and the storage deal size. The content is opaque. After the trigger only
the on-chain actions and the pointer are public; documents stay encrypted until heirs publish.

## Rationale

Executing at block application removes the need for anyone to be online; threshold encryption ensures neither the network
nor a single heir can read the material early; tying pulses to the PQ lock makes "silencing" as hard as stealing the funds.
Automatic pulses from everyday clients (mail, wallet, messenger) remove the main practical failure of dead man's switches -
users forgetting to check in - while keeping the on-chain rule trivially verifiable. Netfory storage keeps the sealed
material available for years without a custodian and without on-chain bloat.

## Security Considerations

Clock is block height (not wall time): a chain halt delays triggers, never fires them early. Owners should choose
`grace ≥ 1 week`. Heir key rotation requires `SwitchUpdate`.

## Backwards Compatibility

New typeGroup / template; Rust-only network.

## Reference Implementation

Not started; prerequisites: [SHIP-33.md](SHIP-33.md) (template execution), [SHIP-37.md](SHIP-37.md) (Netfory storage deals and possession proofs), [SHIP-40.md](SHIP-40.md)
(PQ encryption to heirs), [SHIP-21.md](SHIP-21.md) clients and SmartNet / 0xID Mail automatic pulse support (`switch { deadline }` API).
