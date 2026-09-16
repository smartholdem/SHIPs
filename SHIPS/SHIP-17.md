```
  SHIP: 17
  Title: STH Burn - Fee Burning with Milestone-Controlled Share
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Accepted
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-03-30
  Last Update: 2026-06-19
```

## Abstract

A share of the `TokenInit` fee ([SHIP-14.md](SHIP-14.md)) is permanently removed from circulation by crediting it to the network burn
address inside block application. The share is the milestone parameter `tokenFees.initBurnPercent` (0–100, default 50), so
the network can raise, lower or disable burning without a code release. The mechanism is generic and is the base for the
deflationary token option ([SHIP-28.md](SHIP-28.md)) and for governance decisions ([SHIP-23.md](SHIP-23.md)).

## Motivation

Token issuance consumes network resources forever (registry, symbol, state). Burning part of its fee turns that demand into
deflationary pressure on STH, aligning issuers with holders. Hard-coding the share would make every adjustment a fork.

## Specification

- `network.json -> burnAddress`: an address without a known private key (mainnet `STHsmartHoLdemBurnAddrHereXXXmUW7f`).
  If empty (private test networks), nothing is burned.
- On applying a block: `burned = initCount * tokenFees.init * initBurnPercent / 100`; the forger receives
  `reward + totalFees − burned`; `burned` is credited to `burnAddress` in the same atomic write. No extra transaction is
  produced; rollback reverses it with the block.
- Milestone: `{ "height": H, "tokenFees": { "initBurnPercent": 0 } }` disables burning; `100` burns the whole fee; values
  > 100 are rejected at configuration load. Every `tokenFees` field has a default, so a milestone may patch one key.
- Observability: `GET /api/node/configuration -> tokenFees.initBurnPercent / initBurn`; `GET /api/wallets/<burnAddress>`
  shows the cumulative burned supply; `sth-cli status` prints `token init 500 STH · burn 50% (250 STH)`.

## Rationale

Crediting a provably unspendable address (instead of decrementing a supply counter) keeps the accounting visible with the
existing wallet API and explorer, and the historical `blockBurnAddress` semantics of the network.

## Backwards Compatibility

Applies only to `TokenInit`, which legacy nodes do not accept; gated with SHIP-14.

## Reference Implementation

`src/config.rs` (`TokenFees::init_burn`), `src/storage.rs` / `src/sync.rs` (block fee distribution);
test `tests/tokens.rs::init_burn_percent_switches_by_milestone`.
