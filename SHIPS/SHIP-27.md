```
  SHIP: 27
  Title: Atomic Swaps STH <> Token via HTLC
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-20
  Last Update: 2026-06-19
```

## Abstract

Extend the existing Hashed Time-Locked Contract transactions (`typeGroup 1`, types 8/9/10 - lock, claim, refund) to
**native token amounts**, so two parties can swap STH for a token (or a token for a token) atomically: either both legs
complete with the same secret, or both refund after their timelocks. The same construction enables cross-chain swaps with
any HTLC-capable chain.

## Motivation

The sObject market ([SHIP-16.md](SHIP-16.md)) trades objects for STH at fixed prices; trading token *amounts* peer-to-peer today needs a
trusted intermediary. HTLCs are already in consensus for STH; generalising the locked asset gives trustless OTC trading and
DEX-like order matching off-chain with on-chain settlement.

## Specification

### Token HTLC lock (`typeGroup 3`, `type 13`)

`tokenId 32 ‖ amount u64 ‖ secretHash 32 ‖ expirationType u8 ‖ expiration u32 ‖ recipient 21` - moves `amount` of the
sender's token balance into lock `lk:<txId>` with `asset = { tokenId, amount }`. Fee `tokenFees.htlcLock` (0.1 STH).

### Claim / refund

Existing `HtlcClaim` (`type 9`) and `HtlcRefund` (`type 10`) work unchanged on token locks: claim requires the preimage
(`sha256(secret) == secretHash`) and credits the token to `recipient`; refund after `expiration` returns it to the sender.
Lock ids are shared between STH and token locks; `GET /api/locks/:id` shows `asset`.

### Swap protocol (STH <> token)

1. Alice generates `secret`, locks 1 000 COFFEE for Bob with `sha256(secret)`, expiration `T + 48 h`.
2. Bob sees the lock (`/api/locks?recipientId=Bob`), locks 500 STH for Alice with the **same hash**, expiration `T + 24 h`.
3. Alice claims Bob's STH with `secret` - the preimage becomes public on chain.
4. Bob claims Alice's COFFEE with the revealed `secret`.
5. If either party stalls, refunds return the funds after the expirations (Bob's shorter lock protects him from Alice
   claiming late).

### Rules

Token locks count as neither balance nor vote weight; a frozen account ([SHIP-25.md](SHIP-25.md)) cannot lock or claim that token;
locked token supply is unchanged (`Σ balances + Σ locks = supply`). `expiration` uses the same epoch/height types as STH HTLC.

### Off-chain matching

An order book (wallet plugin or Netfory service, [SHIP-37.md](SHIP-37.md)) publishes signed intents; settlement is always the on-chain HTLC
pair, so the matcher never holds funds.

## Rationale

Reusing claim/refund keeps the trust model identical to the audited STH HTLC; only the lock carries an asset descriptor.

## Backwards Compatibility

New lock type behind the `tokens` milestone; STH HTLC unchanged.

## Reference Implementation

Not started; STH HTLC in `src/rules.rs`, `src/storage.rs` (`lk:`), `src/api/locks.rs`.
