```
  SHIP: 24
  Title: TokenUpdate - Ownership Transfer and Flag Tightening
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-13
  Last Update: 2026-06-19
```

## Abstract

`TokenUpdate` (`typeGroup 3`, `type 5`) lets a token owner (a) hand the token's administrative role to another address
**independently of the ticker sObject**, and (b) change `flags` only in the *permitted direction* - from more power to less
(disable minting, disable freezing), never back. Holders gain guarantees that the issuer can renounce, and issuers gain the
ability to split "brand" (ticker) from "administration".

## Motivation

Since [SHIP-14.md](SHIP-14.md) the token owner is the sObject owner; selling the ticker ([SHIP-16.md](SHIP-16.md)) sells the mint right with it, and there is no
way to make a token provably fixed-supply after launch. Both are common requirements for treasuries, DAOs and regulated
issuers.

## Specification

Payload: `tokenId 32 ‖ newOwner 21 (zero = unchanged) ‖ flags u8 ‖ renounce u8`

- Sender must be the current token owner (`tk:<id>.owner`).
- `flags` may only **clear** bits: `mintable` (bit0), `burnable` may not be cleared (holders' right), `freezable` (bit3,
  [SHIP-25.md](SHIP-25.md)SHIP-25), `deflation` (bit4, [SHIP-28.md](SHIP-28.md)) may be cleared or lowered per that SHIP. Setting a cleared bit is invalid
  (`TokenFlagsCannotBeRaisedError`).
- `newOwner ≠ 0`: `tk:<id>.owner := newOwner`; from then on ownership of the token **no longer follows** the sObject
  (`ownerDetached = true`); the ticker object can still be sold as a name, without administrative power.
- `renounce = 1`: owner := none; token is immutable (no mint, no meta, no further updates); `mintable` must already be 0.
- Fee: `tokenFees.update` (proposed 1 STH). `TokenMeta` ([SHIP-15.md](SHIP-15.md)) follows the token owner, not the sObject, once detached.

API: `tokens[].owner`, `ownerDetached`, `renounced`; CLI `sth-cli tx token-update --to | --flags | --renounce`.

## Rationale

Monotonic flags are the simplest guarantee wallets can display ("supply fixed forever"); detaching ownership keeps [SHIP-16.md](SHIP-16.md)
market semantics for names while letting issuers keep control after a rebrand or sale.

## Backwards Compatibility

New type behind the `tokens` milestone; existing tokens keep coupled ownership until they send a `TokenUpdate`.

## Reference Implementation

Not started. `models::token::UPDATE = 5`, `rules::check_token`, `storage` (`owner_detached`, `renounced`).
