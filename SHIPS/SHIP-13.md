```
  SHIP: 13
  Title: SmartObjects (sObject) - Programmable On-Chain Objects
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-02-02
  Last Update: 2026-06-19
```

## Abstract

A **SmartObject (sObject)** is a named, typed, owned object recorded on the SmartHoldem chain by a single generic transaction
type (`typeGroup 2`, `type 6`). Objects are created, updated and resigned by their owner and - with the `sobjV2` rules
(SHIP-16) - transferred and traded. Type 5 objects are the registries of native tokens (SHIP-14). The generic
`type / subType / action / data` design means new object classes need validation rules, not new transaction types.

## Motivation

Business, product, plugin and token registries all share the same needs: a unique name, an owner, a pointer to external
data and a lifecycle. Modelling each as a dedicated transaction type multiplies serializers, handlers and wallet code.
One object type with a small action set keeps the protocol small and lets applications define semantics in `type`.

## Specification

### Asset

```json
{ "type": 5, "subType": 0, "action": 0,
  "registrationId": "<64 hex>  (actions 1–5)",
  "recipientId":    "<address> (action 3)",
  "price":          "<smartoshi> (action 4)",
  "data": { "name": "COFFEE", "ntfryData": "sth://coffee/manifest.json" } }
```

Wire payload after the common header ([SHIP-11.md](SHIP-11.md)): `u8 type ‖ u8 subType ‖ u8 action ‖ u8 len ‖ registrationId ‖
u8 len ‖ name ‖ u8 len ‖ slot`, where `slot` carries `ntfryData` (actions 0/1), `recipientId` (3) or the decimal `price` (4).
`amount = 0`, fees are fixed per action.

| action | name | fee | rules |
|---:|---|---:|---|
| 0 | register | 50 STH | `(name, type)` unique network-wide (case-insensitive); id = transaction id; owner = sender |
| 1 | update | 5 STH | owner only; replaces `ntfryData`; name immutable |
| 2 | resign | 5 STH | owner only; object frozen, name stays taken; type-5 registries with live token supply cannot resign |
| 3 | transfer | 5 STH | SHIP-16 |
| 4 | sell | 1 STH | SHIP-16 |
| 5 | buy | 1 STH | SHIP-16 |

Types: `4` - delegate object (name must equal the sender's delegate username, never transferable); `5` - token ticker
(`^[A-Z0-9]{3,10}$` under `sobjV2`); `0–3, 6–255` - application-defined.

`ntfryData` (≤ 255 bytes UTF-8 under `sobjV2`; base58 ≤ 128 in legacy rules) is an opaque pointer to Netfory/Web4 content
(SHIP-37); nodes never resolve it. The legacy key `ntfryData` is accepted as an alias on input.

### State

`wallet.attributes.sobjects[registrationId] = { type, subType, data, resigned, price }` in the wallet of the **current**
owner; indexes `en:` (name), `eo:` (owner after transfer), `mk:` (open orders) ([SHIP-3.md](SHIP-3.md)).

### API and tooling

`GET /api/sobj`, `GET /api/sobj/:id`, `POST /api/sobj/search`; wallets expose `attributes.sobjects`;
`sth-cli obj`, `sth-cli tx obj-*` ([SHIP-10.md](SHIP-10.md)). Full specification: `docs/SPEC-SOBJECT.md`.

### Activation

Registration/update/resign rules are active on mainnet since height **11 800 000** (milestone key `SHIP-13`, alias `sobj`).
Milestone `sobjV2` enables UTF-8 pointers, ticker names and actions 3–5; it is scheduled with [SHIP-14.md](SHIP-14.md).

## Rationale

The wire format was kept identical to the legacy entity declaration so that legacy nodes validate the base actions during
the transition; everything above the byte level (naming, API routes, rules v2) is SmartHoldem's own.

## Backwards Compatibility

Wire-compatible with legacy nodes for actions 0–2; actions 3–5 and v2 rules are milestone-gated.

## Reference Implementation

`src/models/transaction.rs` (`sobj`), `src/rules.rs` (`check_sobj_format`, `check_sobj`), `src/storage.rs`,
`src/api/sobj.rs`; tests `tests/sobj.rs`.
