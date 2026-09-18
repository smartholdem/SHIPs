```
  SHIP: 43
  Title: Token Market - Partial Sell Orders for Native Token Balances (On-Chain DEX)
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core / Interface
  Created: 2026-09-19
  Last Update: 2026-09-19
  Requires: 13, 14, 16, 41
```

## Abstract

SHIP-16 lets an owner sell a **SmartObject** - a ticker registry (type 5) and with it the whole token issue - as one
indivisible item. This proposal adds an order book for **token balances**: any holder can place a sell order for *part*
of their balance of a SHIP-14 native token at a fixed STH price per unit ("sell 1 UFO of my 80 for 100 STH"), any buyer
can fill it fully or partially in one atomic transaction, and the seller can cancel the unfilled remainder. Three new
transaction types in `typeGroup 3` (`TokenSell = 5`, `TokenBuy = 6`, `TokenCancel = 7`), an escrow held inside the order
record, deterministic partial fills and a per-token price index give explorers and wallets a DEX without smart
contracts.

## Motivation

Native tokens (SHIP-14) have balances, transfers, mint/burn and metadata, but no on-chain price discovery: a holder who
wants to sell some units must find a counter-party off-chain and trust one side to pay first, or wrap the trade in HTLCs
(SHIP-9) that need two chains of custody per trade. SHIP-16 sale orders cover only the registry object. The Netfory
ecosystem (SHIP-42 provider payments, in-game items, community tokens) needs a simple, trust-less spot market: post an
ask, get paid when someone buys, cancel any time.

## Specification

### Terms

| term          | meaning                                                                                       |
|---------------|-----------------------------------------------------------------------------------------------|
| **order**     | An open ask created by `TokenSell`; identified by the `TokenSell` transaction id (`orderId`). |
| **escrow**    | Token units moved out of the seller's balance into the order record when the order is placed. |
| **price**     | STH (smartoshi) per **one token unit** (smallest unit, i.e. `10^-decimals` tokens).           |
| **remaining** | `order.amount − order.filled`.                                                                |

### Transactions (typeGroup 3)

Common header as every token transaction (SHIP-14 §wire): `tokenId` (32 bytes) first. Amounts are `u64` LE.

| type | name          | payload after `tokenId`   | fee (milestone `tokenFees`)          |
|-----:|---------------|---------------------------|--------------------------------------|
|    5 | `TokenSell`   | `u64 amount ‖ u64 price`  | `tokenFees.sell` (default 1 STH)     |
|    6 | `TokenBuy`    | `32 orderId ‖ u64 amount` | `tokenFees.buy` (default 0.1 STH)    |
|    7 | `TokenCancel` | `32 orderId`              | `tokenFees.cancel` (default 0.1 STH) |

JSON asset (`asset.token`): `{ "id", "amount", "price" }`, `{ "id", "orderId", "amount" }`, `{ "id", "orderId" }`;
`amount`, `price` are decimal strings. `tx.amount = 0` for all three; the STH leg of a buy is derived, never sent in
`tx.amount`. Fees are static per type (exact match, like every typeGroup-3 transaction), plus the PQ surcharge
(SHIP-40).

### Validation

`TokenSell` (sender = seller):

1. token exists (`TokenNotFoundError`); `amount > 0`, `price > 0` (`TokenOrderInvalidError`);
   `amount × price ≤ u64::MAX` (`TokenOrderOverflowError`);
2. `tokenBalance(seller) ≥ amount` after all preceding transactions of the block/pool (`TokenInsufficientBalanceError`);
3. open orders per seller per token ≤ `tokenMarket.maxOpenOrders` (default 64, `TokenTooManyOrdersError`).

`TokenBuy` (sender = buyer):

1. order exists and is open (`TokenOrderNotFoundError` / `TokenOrderClosedError`); `order.tokenId == asset.id`
   (`TokenOrderMismatchError`);
2. `amount > 0` and `amount ≤ remaining` (`TokenOrderAmountError`) - a buy for more than the remainder is invalid,
   wallets MUST clamp;
3. `buyer ≠ seller` (`TokenOrderSelfTradeError`);
4. `balance(buyer) ≥ amount × price + fee` (`InsufficientBalanceError`).

`TokenCancel` (sender = seller): order exists, is open and `order.seller == sender` (`TokenOrderNotOwnerError`).

Within one block several buys may target one order: they are applied in block order; the first that exceeds the current
`remaining` invalidates the block (`TokenOrderAmountError`) - forgers MUST re-check against the running state while
building the block (same rule as token balances in SHIP-14).

### State transitions

```
TokenSell:   seller.tokens[id]  -= amount
             order[orderId] = { tokenId, seller, amount, filled: 0, price, height, closed: null }
TokenBuy:    buyer.balance      -= amount × price          buyer.tokens[id] += amount
             seller.balance     += amount × price          order.filled     += amount
             if order.filled == order.amount: order.closed = height   (fully filled)
TokenCancel: seller.tokens[id]  += remaining                order.closed = height  (cancelled)
```

Escrowed units are counted in the token's `supply` but in **no** wallet balance; `GET /api/tokens/:id/holders` lists
them under the pseudo-holder `"orders"`. A token with `FLAG_FROZEN_CAP`/burnable flags behaves normally; escrowed units
cannot be burned (they are not in a balance).

Order records are never deleted - `closed` marks the end state so that rollback is exact: undoing a buy is
`filled −= amount` (and `closed = null`), undoing a cancel is `closed = null` and `seller.tokens[id] −= remaining`,
undoing a sell removes the record and returns the escrow. Records of orders closed for more than
`tokenMarket.pruneAfterBlocks` (default 1 000 000) MAY be pruned from the live index, never from history.

### Storage and indexes (SHIP-3)

| key                                      | value                                                                 |
|------------------------------------------|-----------------------------------------------------------------------|
| `to:<orderId>`                           | order record (msgpack/JSON)                                           |
| `tob:<tokenId>:<price BE u64>:<orderId>` | open-order book entry, price-ascending - removed when `closed` is set |
| `tos:<seller>:<orderId>`                 | open orders of a seller (for `maxOpenOrders` and wallet views)        |

### API

| method | path                                 | description                                                                                     |
|--------|--------------------------------------|-------------------------------------------------------------------------------------------------|
| GET    | `/api/tokens/:key/orders?page&limit` | open asks, cheapest first: `orderId, seller, price, amount, filled, remaining, height`          |
| GET    | `/api/tokens/:key/orders/:orderId`   | one order incl. `closed` height and `fills` (buy tx ids)                                        |
| GET    | `/api/tokens/:key/trades?page&limit` | executed buys newest first: `txId, buyer, seller, amount, price, total, height`                 |
| GET    | `/api/wallets/:id/orders`            | open orders of a wallet across tokens                                                           |
| GET    | `/api/ntfry/market`                  | extended with `tokenOrders: { open, volume24h }`; the metrics page shows the top asks per token |
| GET    | `/api/node/configuration`            | `constants.tokenMarket` and the three new fee keys                                              |

`sth-cli tx token-sell <TICKER|id> --amount --price`, `token-buy <orderId> --amount`, `token-cancel <orderId>`,
`sth-cli token orders <TICKER>`.

### Activation

Milestone block:

```json
{
  "height": <H>,
  "tokenMarket": {
    "enabled": true,
    "maxOpenOrders": 64,
    "pruneAfterBlocks": 1000000
  },
  "tokenFees": {
    "sell": 100000000,
    "buy": 10000000,
    "cancel": 10000000
  }
}
```

Before `tokenMarket.enabled` the three types are rejected with `TokenTypeNotActiveError`. This is a consensus change:
legacy (Node.js) nodes do not know `typeGroup 3` at all, so it is safe only on a network whose forging set runs
`sth-core`
(same condition as SHIP-14).

## Rationale

* **Escrow in the order, not in the wallet.** Moving units out of the balance at placement time makes every later check
  a plain balance/remaining comparison - no "reserved" sub-balance to keep consistent across transfers, burns and
  rollbacks.
* **Fixed-price asks only, no bids, no matching engine.** A buy names the order it fills, so execution is deterministic
  and independent of block ordering across nodes; bids and automatic matching can be layered later without changing
  these types.
* **Price per smallest unit as `u64`.** Avoids fractions in consensus; wallets display `price × 10^decimals` per token.
* **Never-deleted order records.** Exact rollback and full trade history for explorers at the cost of one small record
  per order.
* **Static fees.** Consistent with [SHIP-14.md](SHIP-14.md) and with the fixed sObject action fees; dynamic fees ([SHIP-41.md](SHIP-41.md)) stay a
  typeGroup-1 policy.

## Backwards Compatibility

Additive: existing token types and [SHIP-16.md](SHIP-16.md) registry sales are unchanged; `/api/tokens/*` gains new sub-resources only.
Wallets that do not know the new types show them as generic typeGroup-3 transactions (`/api/transactions/types` lists
`TokenSell: 5, TokenBuy: 6, TokenCancel: 7`).

## Reference Implementation

Implemented in PQ node `sth-core-rust` 0.20 (`tests/tokens.rs::token_market_orders_fill_cancel_and_rollback`); the plan
was: `models::token::{SELL, BUY, CANCEL}` + `TokenAsset { order_id, price }`, serializer / deserializer cases,
`rules::check_token_format` / `check_token` (`TokenView` gains `order(&str)` and
`open_orders(seller, token)`), `storage` order keys + `accumulate_deltas` + rollback, `sync::BatchTokenView` in-batch
orders, `api::tokens::{orders, order, trades}`, `sth-cli` commands, suites `tests/token_market.rs` (place / partial
fill / over-fill rejection / cancel / rollback) and an `api` case for the order book.

## Security Considerations

* **Front-running.** Asks are fixed-price; a forger can order buys but cannot change prices - the worst case is choosing
  which of two buyers fills the remainder, bounded by one block.
* **Dust orders / spam.** `tokenFees.sell` (1 STH) plus `maxOpenOrders` per seller per token; explorers SHOULD hide asks
  whose total value is below the buy fee.
* **Overflow.** `amount × price` checked in `u128` at validation; the STH leg is a plain balance transfer inside the
  same atomic state update as the token leg.
* **Self-trade / wash trading.** Self-buys are invalid; wash trading between two wallets is possible, as on any DEX, and
  costs fees on both sides.
* **Escrow safety.** Units in escrow are unreachable by any transaction except the fill/cancel rules above; a token
  owner cannot burn or freeze them.

## Copyright

This document is placed in the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
