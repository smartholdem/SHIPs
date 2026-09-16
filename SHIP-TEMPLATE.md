```
  SHIP: <to be assigned by the editors>
  Title: <Short, descriptive title (≤ 60 characters)>
  Authors: <Name <email>>, <Name <email>>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues/<number>
  Type: <Standards Track | Informational | Process>
  Category: <Core | Networking | Interface | Tooling>   (Standards Track only)
  Created: <YYYY-MM-DD>
  Last Update: <YYYY-MM-DD>
  Requires: <SHIP numbers this proposal depends on>     (optional)
  Replaces: <SHIP number>                               (optional)
```

## Abstract

A short (~200 word) description of the technical issue being addressed. What does the proposal change, for whom, and
what is the one-sentence outcome?

## Motivation

Why is the existing protocol, node behaviour or process inadequate? Describe the problem, who experiences it and why it
cannot be solved without this SHIP. Proposals without a clear motivation are rejected.

## Specification

The technical specification, detailed enough for independent, interoperable implementations. Include as applicable:

- **Transactions / wire format** - `typeGroup`, `type`, payload layout (byte table), JSON `asset` shape, fees.
- **Rules** - stateless checks, wallet-aware checks, error names (`ERR_*` / `*Error`), block-internal ordering effects.
- **State** - Sled key prefixes and `WalletState` attributes touched (see [SHIP-3.md](SHIPS/SHIP-3.md)), undo / rollback behaviour.
- **Milestone** - activation flag(s) and parameters, e.g. `{ "height": H, "feature": { "active": true, ... } }`.
- **API** - new or changed endpoints and fields (`/api/...`), metrics page changes (see [SHIP-8.md](SHIPS/SHIP-8.md)).
- **Tooling** - `sth-cli` / `sth-core` commands.

Use tables for layouts and limits; use exact numbers (bytes, fees in smartoshi and STH, heights).

## Rationale

Why this design and not the alternatives? Record the alternatives considered and the reasons for rejecting them, plus
any related work in other networks.

## Benefits

What operators, delegates, wallets and users gain (measurable where possible: bytes, ms, STH).

## Backwards Compatibility

Does the change require all 21 active delegates on a new core version (milestone gate)? Does it affect legacy nodes,
existing wallets, explorers, exchanges or stored databases? How is the transition handled?

## Security Considerations

Threats introduced or mitigated, failure modes, key-loss or irreversibility warnings, denial-of-service surface
(payload sizes, verification cost).

## Reference Implementation

Module paths, tests and vectors in `sth-core` (or "Not started" with the planned touch points).

## Copyright

Copyright and related rights waived via CC0.
