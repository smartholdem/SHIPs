```
  SHIP: 15
  Title: Token Manifest (TokenMeta) with On-Chain Logos
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-03-02
  Last Update: 2026-07-10
```

## Abstract

`TokenMeta` (`typeGroup 3`, `type 4`) stores a token's human-readable manifest - name, description, website and a logo
(SVG or PNG, ≤ 8 192 bytes) - **inside the chain**, so every wallet, explorer and market shows the same verified identity
without trusting an external server or a token list.

## Motivation

Token lists maintained off-chain are centralised, go stale, and are a phishing vector (look-alike tickers with copied
logos). Putting the manifest under the owner's signature in consensus makes identity part of the asset and available to
every node's API.

## Specification

Payload: `tokenId 32 ‖ nameLen u8 ‖ name ‖ descLen u16 ‖ description ‖ siteLen u8 ‖ website ‖ logoType u8 ‖ logoLen u16 ‖ logo`

| field | limit |
|---|---|
| `name` | ≤ 64 bytes UTF-8 |
| `description` | ≤ 512 bytes |
| `website` | ≤ 128 bytes, `https://` |
| `logoType` | 0 none · 1 SVG (`image/svg+xml`, must parse as XML, no `<script>`/external refs) · 2 PNG (valid signature/IHDR) |
| `logo` | ≤ 8 192 bytes |

Only the current registry owner may send it; fee `tokenFees.meta` (1 STH); re-sending replaces the manifest entirely.
The manifest is stored in `tk:<tokenId>.meta`, served as `meta` in `/api/tokens/*` and as an image at
`GET /api/tokens/:id/logo` (correct `Content-Type`, immutable caching by block height). The metrics page and the market tab
render logos directly from the node.

Security: SVG is sanitised on validation (rejected if it contains scripts, event handlers, external hrefs); clients must
still render it in an isolated `<img>` context.

## Rationale

8 KB fits a crisp SVG or a 128 px PNG while keeping the transaction under the gossip message limit; a full replace
semantic avoids partial-update ambiguity.

## Reference Implementation

`src/models/transaction.rs` (`TokenMeta`), `src/rules.rs`, `src/api/tokens.rs` (`logo`); tests `tests/tokens.rs`;
docs `docs/WALLET-TOKEN-META.md`.
