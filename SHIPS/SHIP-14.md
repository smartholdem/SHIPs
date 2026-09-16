```
  SHIP: 14
  Title: Native Tokens (typeGroup 3)
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-02-16
  Last Update: 2026-06-19
```

## Abstract

Fungible tokens implemented **natively in consensus** - no virtual machine, no per-token code: a token is a sObject of type 5
(its ticker, [SHIP-13.md](SHIP-13.md)) plus a fixed set of transactions in `typeGroup 3` (init, transfer, mint, burn, meta). Balances live in
wallet state, transfers cost microseconds, and every token has the same audited rules.

## Motivation

Contract-based tokens re-implement the same logic per token, each with its own bugs and gas costs. SmartHoldem already has
fast, deterministic state transitions for STH; giving tokens the same engine yields transfers as cheap as STH transfers
(~0.09 ms to apply) with multi-recipient support and a uniform explorer/wallet experience.

## Specification

### Registry

`tokenId = registrationId` of the type-5 sObject; the token owner **is** the sObject owner (moves with transfer / buy,
[SHIP-16.md](SHIP-16.md)). A resigned registry cannot init a token; a registry with live supply cannot resign (resign guard).

### Transactions (`typeGroup 3`, `amount = 0`, version 2 or 3)

| type | name | who | payload (after header) | fee (milestone `tokenFees`) |
|---:|---|---|---|---|
| 0 | TokenInit | registry owner | `tokenId 32 ‖ decimals u8 ‖ flags u8 ‖ initialSupply u64 ‖ supplyCap u64` | `init` = 500 STH, `initBurnPercent` (50 %) burned (SHIP-17) |
| 1 | TokenTransfer | holder | `tokenId ‖ count u16 ‖ (amount u64 ‖ recipient 21)* ‖ memoLen u8 ‖ memo` | `transfer` 0.1 STH + `transferPerRecipient` 0.01 STH × (count − 1) |
| 2 | TokenMint | owner | `tokenId ‖ amount u64 ‖ recipient 21` | `mint` 1 STH; requires `mintable`, ≤ `supplyCap` |
| 3 | TokenBurn | holder | `tokenId ‖ amount u64` | `burn` 0.1 STH; requires `burnable` |
| 4 | TokenMeta | owner | SHIP-15 | `meta` 1 STH |

`flags`: bit0 `mintable`, bit1 `burnable`, bit2 `frozenCap` (reserved, always set), bits 3–7 reserved (SHIP-24/25/28 will
assign them). `decimals` 0–18; amounts are `u64` strings in minimal units; `memo` ≤ 64 bytes UTF-8;
`count` ≤ `tokenTransferMaxRecipients` (64). Fees are **exact** (`StaticFeeMismatchError`), plus the v3 surcharge when the
sender is PQ-locked ([SHIP-19.md](SHIP-19.md)).

### State

`tk:<tokenId> → { symbol, decimals, flags, supply, supplyCap, owner, initHeight, meta }`; `tks:<SYMBOL> → tokenId`;
`wallet.attributes.tokens[tokenId] = balance`. All updates are atomic per block with undo ([SHIP-3.md](SHIP-3.md)).

### Rules

Sender balance ≥ sum of outputs; recipients distinct from nothing (self-transfer allowed); supply invariants
`Σ balances = supply ≤ supplyCap`; mint only by the current owner; STH fee paid in STH with strict balance check.

### API

`GET /api/tokens`, `/api/tokens/:idOrSymbol`, `/api/tokens/:id/holders`, `/api/tokens/:id/logo`, wallet `attributes.tokens`,
`GET /api/node/configuration → tokenFees`. CLI: `sth-cli tx token-init | token-transfer | token-mint | token-burn`.

### Activation

Milestone `{ "height": H_TOKENS, "tokens": true, "sobjV2": true, "strictBalance": true }` - scheduled when all 21 active
delegates run `sth-core` ([SHIP-1.md](SHIP-1.md)). Full specification: `docs/SPEC-TOKENS-NATIVE.md`.

## Rationale

Tying ownership to the sObject reuses naming, uniqueness and ([SHIP-16.md](SHIP-16.md)) the market for free; fixed fees paid in STH keep the
fee market simple and make token activity a demand driver for STH.

## Backwards Compatibility

Legacy nodes reject `typeGroup 3`; hence the milestone gate.

## Reference Implementation

`src/models/transaction.rs` (`token`), `src/rules.rs` (`check_token_format`, `check_token`), `src/storage.rs`,
`src/api/tokens.rs`; tests `tests/tokens.rs`, `tests/balance.rs`.
