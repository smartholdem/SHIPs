```
  SHIP: 41
  Title: Dynamic Fees - Per-Node Pool Policy and the Wallet Fee Formula
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Interface
  Created: 2026-09-16
  Last Update: 2026-09-17
  Requires: 2, 11
```

## Abstract

Defines how a SmartHoldem node decides which fee a **core** transaction (typeGroup 1) needs to enter its pool and be
relayed, and - the part that matters for users - how every wallet must compute the fee so that a transaction built against
any node is accepted by every other node. Dynamic fees are a **node policy**, not a consensus rule: blocks are never
rejected because of a transaction fee. The formula is the legacy one (`(addonBytes[type] + wireBytes) × satoshiPerByte`),
so `sth-core` and legacy nodes agree byte-for-byte.

## Motivation

Static fees on mainnet are 1 STH per transfer; the legacy nodes have run with `dynamicFees.enabled = true` for years and a
plain transfer actually costs ≈ 0.0077 STH (`minFeePool = 3000` smartoshi/byte). Two problems appeared with the Rust
rollout:

1. `sth-core` < 0.20 reported `dynamicFees.enabled = false`, so wallets connected to a PQ Nodes charged users the static
   1 STH - 130x more than through a legacy node - while its pool accepted *any* fee (even 1 smartoshi), which opened the door
   to spam.
2. The difference between "enter the pool" and "be broadcast" was folklore.

This SHIP fixes the formula, the constants and the API surface so that all wallets, explorers and nodes compute the same
minimum.

## Specification

### Terms

- `wireBytes` - length in bytes of the serialised transaction ([SHIP-11.md](SHIP-11.md) v2 or [SHIP-19.md](SHIP-19.md) v3 format, **including** signatures).
  For the fee check the pool uses the transaction exactly as received; for wallets the table below gives the sizes.
- `addonBytes[type]` - virtual extra weight per core type (spam pricing), see the table.
- `rate` - smartoshi per byte: `minFeePool` to enter the pool, `minFeeBroadcast` to be relayed.

### Formula

```
minFee(type, wireBytes, rate) = (addonBytes[type] + wireBytes) × max(rate, 1)
```

A node with `dynamic_fees.enabled = true` (default):

| Condition | Result | API answer |
|---|---|---|
| `fee < minFee(type, bytes, minFeePool)` | rejected | `ERR_LOW_FEE`, message contains the minimum |
| `minFee(minFeePool) ≤ fee < minFee(minFeeBroadcast)` | accepted, kept local | listed in `accept`, **not** in `broadcast` |
| `fee ≥ minFee(type, bytes, minFeeBroadcast)` | accepted and relayed | `accept` + `broadcast` |

A node with `dynamic_fees.enabled = false` accepts a core transaction only when `fee == staticFees[type]` of the current
milestone (legacy behaviour); anything else is `ERR_LOW_FEE`.

Not covered by this SHIP (their fees are exact and consensus-checked): sObject transactions ([SHIP-13.md](SHIP-13.md), [SHIP-16.md](SHIP-16.md)), token
transactions ([SHIP-14.md](SHIP-14.md), `tokenFees`), v3 post-quantum transactions ([SHIP-20.md](SHIP-20.md): `staticFee + feePerByte × pqBytes`).

### Mainnet constants (identical to the legacy `transactionPool.dynamicFees`)

| type | name                 | addonBytes |
|-----:|----------------------|-----------:|
|    0 | transfer             |        100 |
|    1 | secondSignature      |        250 |
|    2 | delegateRegistration |    400 000 |
|    3 | vote                 |        100 |
|    4 | multiSignature       |        500 |
|    5 | ntfry                |        250 |
|    6 | multiPayment         |        500 |
|    7 | delegateResignation  |        100 |
|    8 | htlcLock             |        100 |
|    9 | htlcClaim            |          0 |
|   10 | htlcRefund           |          0 |

`minFeePool = 3000`, `minFeeBroadcast = 3000` smartoshi/byte. Unknown types have `addonBytes = 0`.

### Wire sizes wallets may assume (v2, Schnorr 64-byte signature, no vendorField)

| type                                      |               bytes | formula                                                           |                                                                                minimum at 3000/byte |
|-------------------------------------------|--------------------:|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------:|
| transfer                                  |                 156 | 59 header + 8 amount + 4 expiration + 21 recipient + 64 signature |                                                      (100 + 156) × 3000 = **768 000** (0.00768 STH) |
| vote (1 vote)                             |                 158 | 59 + 1 count + 34 vote + 64 signature                             |                                                                        (100 + 158) × 3000 = 774 000 |
| delegateRegistration                      | 124 + len(username) | 59 + 1 length + username + 64 signature                           | ≈ (400 000 + 130) × 3000 ≈ 12.0 STH (static fee 10 000 STH is also accepted - any fee ≥ minimum is) |
| delegateResignation                       |                 123 | 59 + 64 signature (no payload)                                    |                                                                        (100 + 123) × 3000 = 669 000 |
| multiPayment (N recipients)               |          125 + 29·N | 59 + 2 count + 29 per recipient + 64 signature                    |                                                                           (500 + 125 + 29·N) × 3000 |
| vendorField adds `len(vendorField)` bytes |                     |                                                                   |                                                                                     + 3000 per byte |

A wallet **should** query the node rather than hard-code the table: `GET /api/node/configuration` →
`data.transactionPool.dynamicFees { enabled, minFeePool, minFeeBroadcast, addonBytes }`. If `enabled` is `false` it must use
`data.constants.fees.staticFees[type]`. `GET /api/node/fees` (legacy) and `GET /api/ntfry/metrics` →
`mempool.dynamicFees.minTransferFee` give the ready-made transfer minimum.

Recommended wallet behaviour: compute `minFee(type, bytes, minFeeBroadcast)` and add a small safety margin (e.g. 10 %,
or round up to 0.01 STH) - a fee between the pool and broadcast thresholds is accepted by the node it was sent to but not
relayed, so the transaction waits until that node's delegate forges.

### Node configuration (`node.yaml`)

```yaml
mempool:
  dynamic_fees:
    enabled: true
    min_fee_pool: 3000
    min_fee_broadcast: 3000
    addon_bytes: { transfer: 100, vote: 100, delegateRegistration: 400000, ... }
```

Operators may raise `min_fee_pool` on an overloaded node; they should not lower it below the network's common value, or
their pool fills with transactions no other node relays.

### Observability

`GET /api/ntfry/metrics` → `mempool.dynamicFees { enabled, minFeePool, minFeeBroadcast, addonBytes, minTransferFee,
transferBytes, lowFeeRejected }`; the metrics page shows `min transfer 0.0077 STH (dynamic) · N low-fee rejected` and the
payout calculator uses the dynamic minimum.

## Rationale

- The formula and constants are the legacy ones so that mixed Rust/legacy networks share one fee market; changing the
  constants (e.g. raising `minFeePool`) is an operator decision per node and needs no fork.
- Two thresholds (pool vs broadcast) are kept for legacy parity, although mainnet sets them equal; wallets that target the
  broadcast threshold are always safe.
- The pool checks the fee before the signature: it is the cheapest filter and the same order as the legacy processor.

## Backwards Compatibility

Fully compatible: no block, transaction or wire-format change. Legacy nodes and `sth-core` ≥ 0.20 compute identical
minimums; transactions built for one are accepted by the other. `sth-core` < 0.20 accepted lower fees than allowed here -
such transactions, once in a block, remain valid (fees are not a consensus rule).

## Reference Implementation

`sth-core` ≥ 0.20.0: `node_config::DynamicFeesConfig` (`min_fee()`), `Mempool::check_core_fee` (pool / broadcast /
static), `Mempool::min_transfer_fee` (`TRANSFER_WIRE_BYTES = 156`), `Mempool::low_fee_rejected`, API
`configuration.transactionPool.dynamicFees`, `metrics.mempool.dynamicFees`. Wallet reference: `sth-cli tx` (`cli::dynamic_fee`,
rounds the minimum up to 0.001 STH, `--fee` override). Tests: `tests/dynamic_fees.rs`.

## Security Considerations

Fees are the only spam brake for core transactions. A node that disables the check (or sets `min_fee_pool` to 0)
exposes only itself: its pool may fill, but other nodes relay nothing below their own minimum and blocks are limited by
`block.maxTransactions` / `maxPayload`. The byte-based formula makes large transactions (multipayments, long vendor fields)
pay proportionally, so the pool byte budget ([SHIP-20.md](SHIP-20.md) implementation) cannot be exhausted cheaply.

## Copyright

Copyright and related rights waived via CC0.
