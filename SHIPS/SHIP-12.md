```
  SHIP: 12
  Title: Multipayment with up to 1024 Recipients
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-01-19
  Last Update: 2026-06-25
```

## Abstract

Raise the multipayment recipient limit (`multiPaymentLimit`, milestone) from 256 to **1 024** once the network runs on
[PQ Nodes](https://github.com/smartholdem/sth-core-pq) only, and define the accompanying block payload and fee rules so that one transaction can settle a payroll,
an airdrop or an exchange batch in a single 8-second slot.

## Motivation

A multipayment costs one signature check (~0.24 ms) regardless of recipient count; the per-recipient cost is state
application (~0.09 ms per operation, [SHIP-7.md](SHIP-7.md) measurements). With 256 recipients the signature is already < 1 % of the cost,
so the limit is a policy choice, not a performance one. Legacy nodes cannot validate larger payments; [PQ Nodes](https://github.com/smartholdem/sth-core-pq) can apply
1024 recipients in ~90 ms.

## Specification

- Milestone: `{ "height": H, "multiPaymentLimit": 1024 }`; wire format unchanged (`u16 count`).
- Fee: `staticFees.multiPayment` + `multiPaymentPerRecipient × (count − 1)` (new milestone key, proposed 0.001 STH), so a
  1 024-recipient payment costs ~1 STH + 1.02 STH.
- Block payload: `block.maxPayload` grows to 4 MB; `maxTransactions` unchanged (150) - worst case 150 × 29.8 KB = 4.4 MB is
  bounded by `maxPayload`, so a block holds at most ~135 full-size multipayments (≈ 138 000 operations per slot).
- Recipient list must not contain duplicates of the sender? - no: allowed (legacy behaviour), but each recipient counts.
- Mempool: per-transaction size limit 32 KB; byte budget (SHIP-20 implementation) already caps total pool bytes.

### Throughput matrix (from `docs/MULTIPAY-1024.md`)

| slot | ops/block at T1 (11 000 op/s) | ops/s |
|---:|---:|---:|
| 8 s | 41 000 | 5 100 |
| 4 s | 19 000 | 4 800 |
| 2 s | 8 000 | 4 000 |

## Rationale

1 024 keeps a full transaction under 30 KB (fits one gossip message) and one block's apply time under 25 % of an 8-second
slot on the reference hardware. Larger limits (2 048+) wait for parallel state application ([SHIP-36.md](SHIP-36.md)).

## Backwards Compatibility

Requires all active delegates on `sth-core` (legacy nodes reject payments above 256).

## Reference Implementation

Pending: milestone key `multiPaymentPerRecipient`, `rules.rs` fee check, `bench_block` extension.
