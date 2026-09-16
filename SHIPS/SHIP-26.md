```
  SHIP: 26
  Title: Non-Fungible Tokens (subType 1: tokenId + serial)
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-18
  Last Update: 2026-06-19
```

## Abstract

NFTs as a **collection** (a type-5 sObject with `subType = 1`) whose items are `(tokenId, serial)` pairs with per-item
metadata pointers; minting, transfer, burn and market listing reuse the native token engine ([SHIP-14.md](SHIP-14.md)) and the sObject
market ([SHIP-16.md](SHIP-16.md)), so an NFT costs the same to move as a token transfer and needs no contract.

## Motivation

Collectibles, tickets, certificates and game items need unique, ownable, tradable records with verifiable provenance.
Contract-based NFT standards re-implement transfer logic per collection; a native, collection-scoped item registry is
cheaper, uniform and inherits Quantum Shield protection of the owner wallet.

## Specification

### Collection

sObject `type 5, subType 1` (ticker = collection symbol) initialised with `TokenInit` where `decimals = 0`,
`flags` bit5 `nft` set, `initialSupply = 0`, `supplyCap = max items (0 = unlimited)`. Fungible rules that do not apply
(divisible amounts) are disabled by the flag.

### Items

- `NftMint` (`typeGroup 3`, `type 7`): `tokenId 32 ‖ serial u64 ‖ recipient 21 ‖ ntfryDataLen u8 ‖ ntfryData (≤ 255)` - owner
  only; `serial` unique within the collection, assigned by the minter (sequential recommended); `supply += 1 ≤ supplyCap`.
- `NftTransfer` (`type 8`): `tokenId ‖ serial ‖ recipient 21 ‖ memoLen u8 ‖ memo` - current item holder only.
- `NftBurn` (`type 9`): `tokenId ‖ serial` - holder; `supply −= 1` (serial never reused).
- `NftUpdate` (`type 10`, optional per collection flag `mutable`): owner updates an item's `ntfryData`.
- Fees: mint 1 STH, transfer 0.1 STH, burn 0.1 STH, update 0.5 STH (`tokenFees.nft*`).

### State

`nft:<tokenId>:<serial BE> -> { owner, ntfryData, mintHeight }`; `wallet.attributes.nfts[tokenId] = [serials]` (bounded
list, paginated API). Item metadata (`ntfryData`) points to a Netfory/Web4 document ([SHIP-37.md](SHIP-37.md)) - image, attributes, license;
the collection manifest is the `TokenMeta` ([SHIP-15.md](SHIP-15.md)).

### Market

Items are listed and bought through sObject-style orders extended with a serial: `NftSell` / `NftBuy` (`type 11 / 12`,
`price` in STH), same atomic settlement as [SHIP-16.md](SHIP-16.md); `GET /api/ntfry/market?nft=1`.

### API

`GET /api/nft/:tokenId` (collection), `/api/nft/:tokenId/:serial`, `/api/wallets/:addr/nfts`.

## Rationale

Numeric serials (not free-form ids) keep keys fixed-size and sortable; collection-level ownership via the ticker sObject
gives creators the same tools as token issuers (manifest, renounce via [SHIP-24.md](SHIP-24.md)).

## Backwards Compatibility

New types behind the `tokens` milestone; wallets that ignore `nft` flag tokens are unaffected.

## Reference Implementation

Not started.
