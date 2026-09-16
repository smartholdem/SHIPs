```
  SHIP: 25
  Title: TokenFreeze - Account Freezing for Regulated Assets
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-15
  Last Update: 2026-06-19
```

## Abstract

An optional, opt-in-at-issuance capability: tokens created with the `freezable` flag allow their owner to freeze and unfreeze
individual holder balances of **that token only**. Frozen balances cannot be transferred or burned by the holder. The
capability can be renounced irreversibly ([SHIP-24.md](SHIP-24.md)); it never exists for tokens issued without the flag.

## Motivation

Security tokens, stablecoins and tokenised deposits must comply with sanctions and court orders; without a native freeze
issuers either avoid the chain or wrap tokens in custodial layers. Making the capability explicit at issuance keeps
"can this asset be frozen?" a verifiable, permanent property every wallet can display.

## Specification

- `flags` bit3 `freezable` - may be set only in `TokenInit`; may be cleared later ([SHIP-24.md](SHIP-24.md)), never set afterwards.
- `TokenFreeze` (`typeGroup 3`, `type 6`): `tokenId 32 ‖ account 21 ‖ freeze u8 (1 freeze / 0 unfreeze) ‖ reasonLen u8 ‖ reason (≤ 128 bytes, e.g. case reference)`.
  Sender = token owner; token must be `freezable`; fee `tokenFees.freeze` (proposed 1 STH).
- State: `wallet.attributes.tokensFrozen[tokenId] = height`; transfers **from** a frozen account (`TokenTransfer`,
  `TokenBurn`) fail with `TokenAccountFrozenError`; transfers **to** it succeed (funds can arrive, not leave).
  Freezing does not affect STH or other tokens of the account.
- Owner may not freeze itself; unfreeze restores the balance's mobility; frozen state is undo-safe and shown in
  `GET /api/wallets/:addr → attributes.tokensFrozen` and `GET /api/tokens/:id/holders` (`frozen: true`).
- Governance interplay: none - the network never freezes; only the issuer of a freezable token can, and only that token.

## Rationale

Per-account freezing (not global pause) matches regulatory practice while keeping issuer power narrow and auditable
(every freeze is a public transaction with a reason field). Tokens without the flag are provably censorship-resistant.

## Security Considerations

Wallets must show the `freezable` badge prominently; markets should treat freezable and non-freezable tokens as different
risk classes.

## Backwards Compatibility

New type and flag behind the `tokens` milestone.

## Reference Implementation

Not started. `models::token::FREEZE = 6`, `FLAG_FREEZABLE = 8`, `rules::check_token`, `storage` (`tokens_frozen`).
