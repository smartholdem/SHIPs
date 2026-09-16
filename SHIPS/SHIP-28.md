```
  SHIP: 28
  Title: Deflationary Tokens - Burn Share on Transfer
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-22
  Last Update: 2026-06-19
```

## Abstract

A token may declare at issuance a **burn share** applied to every transfer: a fixed per-mille of each transferred amount is
destroyed (supply decreases) instead of reaching the recipient. The share is a token parameter, may only be lowered later
([SHIP-24.md](SHIP-24.md)), and is enforced by consensus so wallets can display the exact net amount before sending.

## Motivation

Deflationary mechanics are a popular token design; on contract chains they are implemented per token with frequent bugs
(rounding, exemptions) and hidden fees. A native, declared, monotone-decreasing burn share is transparent and cheap.

## Specification

- `TokenInit` gains `burnPerMille u16` (0–500, i.e. ≤ 50 %) placed after `supplyCap` when `flags` bit4 `deflation` is set
  (payload stays backward-compatible: the field exists only with the flag).
- On `TokenTransfer` per recipient: `burn = floor(amount × burnPerMille / 1000)`; recipient receives `amount − burn`;
  `supply −= burn`. The sender is debited `amount`. Burned units are gone (no burn address for tokens; supply is the ledger).
- Exemptions: none in consensus (owner and market settlements burn too) - simplicity beats special cases; issuers who want
  exemptions can pre-fund.
- `TokenUpdate` ([SHIP-24.md](SHIP-24.md)) may lower `burnPerMille` (including to 0) never raise it; the `deflation` flag cannot be set later.
- Mint (`TokenMint`) is not burned; `TokenBurn` unchanged.
- API: `tokens[].burnPerMille`, per-transfer `burned` in transaction JSON; `sth-cli token` shows `deflation 2.5 % per transfer`.

### Interaction with STH burn ([SHIP-17.md](SHIP-17.md))

Independent: STH fees are burned by network policy; token deflation is the issuer's policy on its own supply.

## Rationale

Per-mille granularity with a 50 % cap covers real designs and prevents "transfer burns everything" traps; recipient-side
burn keeps the sender's debit equal to the declared amount, which is what wallets show.

## Backwards Compatibility

Flag-gated field within `TokenInit`; tokens without the flag are unaffected.

## Reference Implementation

Not started. `models::token::FLAG_DEFLATION = 16`, `rules::check_token` (transfer path), `storage` (`burn_per_mille`).
