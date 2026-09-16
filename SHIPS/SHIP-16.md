```
  SHIP: 16
  Title: sObject Transfer and Decentralised Market (sell / buy)
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-03-16
  Last Update: 2026-07-11
```

## Abstract

Three actions extend SmartObjects ([SHIP-13.md](SHIP-13.md)) under the `sobjV2` milestone: **transfer** (action 3) hands an object to another
address, **sell** (action 4) opens a fixed-price order, **buy** (action 5) settles it atomically - coins to the owner, object
(and, for type 5, the token registry) to the buyer - in a single transaction without escrow, intermediaries or contracts.

## Motivation

Names, tickers and token registries have value; without transfer they are stuck with the registrant forever, and any sale
requires trusting a counterparty. An on-chain order settled by consensus removes that trust and gives wallets a native
marketplace for every object class.

## Specification

| action | fee | sender | fields | effect |
|---:|---:|---|---|---|
| 3 transfer | 5 STH | owner | `registrationId`, `recipientId` | owner := recipient; open order cancelled; type 4 (delegate) never transferable; not to self |
| 4 sell | 1 STH | owner | `registrationId`, `price` (decimal string, smartoshi) | `price > 0` opens/updates the order; `price = 0` cancels |
| 5 buy | 1 STH | anyone but the owner | `registrationId` | requires open order; buyer pays `price + fee`; `price` credited to the owner; object and token registry move; order closed |

Wire: `recipientId` / `price` travel in the generic `slot` field of the sObject payload, so blocks keep the legacy byte layout.
Rules are state-aware within a block (A->B->C in one block is valid; the old owner immediately loses rights). Resigned
objects cannot be transferred or sold. Mempool reserves `price` of a pending buy for the buyer's later transactions.

State: object record moves between `attributes.sobjects` of the two wallets; `eo:` records the owner, `mk:` indexes open
orders; token registry `tk:<id>.owner` follows the object. Rollback restores all three.

API: `GET /api/ntfry/market?type=5` (orders with token manifests), `tokens[].sobj.{price, forSale}`, metrics page tab
**market**; CLI `obj-transfer | obj-sell | obj-buy`.

## Rationale

Fixed-price orders cover the dominant use case (selling a name/ticker) with one state field; auctions and bids can be added
as further actions without changing the wire format.

## Backwards Compatibility

Actions 3–5 are rejected before `sobjV2` (`SmartObjectTransferNotActiveError`); the milestone is scheduled with [SHIP-14.md](SHIP-14.md).

## Reference Implementation

`src/rules.rs` (`check_sobj`), `src/storage.rs` (market index, ownership move), `src/api/ntfry.rs` (`market`),
`src/mempool.rs` (`spent`); tests `tests/sobj.rs`.
