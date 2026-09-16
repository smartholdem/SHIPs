```
  SHIP: 23
  Title: On-Chain Governance - Delegate Proposals with 11-of-21 Quorum
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-11
  Last Update: 2026-08-10
```

## Abstract

A DAO process in consensus: any active delegate may publish a **proposal** that changes a whitelisted set of milestone
parameters (fees, burn share, block reward, feature flags); the 21 active delegates vote on-chain; a proposal that reaches
**11 of 21** approvals becomes a scheduled milestone that activates **no earlier than 2 weeks** after publication, during
which token holders can re-vote their delegates and the proposal can be rejected. Relay nodes apply accepted parameter
changes automatically - no binary release, no coordinator.

## Motivation

Today parameter changes are edits to `milestones.json` shipped with a core release: slow, centralised around maintainers and
invisible to voters until it happens. The network already has an elected body (21 delegates) and a voting mechanism (vote
transactions); governance should reuse both and give holders a veto window.

## Specification

### Transactions (`typeGroup 4`)

| type | name | who | payload |
|---:|---|---|---|
| 0 | Proposal | active delegate | `kind u8 ‖ activationHeight u64 ‖ payloadLen u16 ‖ payload (JSON patch of allowed keys) ‖ titleLen u8 ‖ title ‖ ntfryData (≤ 255, rationale document)` |
| 1 | ProposalVote | active delegate | `proposalId 32 ‖ approve u8` (one vote per delegate, may be changed) |
| 2 | ProposalCancel | proposer | `proposalId 32` |

Fees: proposal 100 STH (burned 100 %), vote 1 STH, cancel 1 STH.

### Allowed parameters (whitelist, `kind = 0` parameter patch)

`fees.staticFees.*`, `tokenFees.*` incl. `initBurnPercent` ([SHIP-17.md](SHIP-17.md)), `reward` (block reward on/off or value within
`[0, 2 × current]`), `multiPaymentLimit` (≤ 1024, [SHIP-12.md](SHIP-12.md)), `tokens`, `sobjV2`, `pq.active`, `pq.feePerByte`,
`minCoreVersion`. Anything else is invalid. `kind = 1` (text-only signal) has no effect.

### Timing

- `activationHeight ≥ publicationHeight + 151 200` (14 days at 8 s) and `≤ + 453 600` (42 days).
- Voting is open from publication until `activationHeight − 10 800` (1 day before). Status at that moment:
  **approved** if approvals from ≥ 11 delegates **who are active at that height**; otherwise **rejected**.
- Because the active set is recomputed every round, holders re-voting delegates during the window changes who counts -
  this is the holder veto. A delegate that leaves the top 21 loses its vote.

### Activation

At `activationHeight` every node deep-merges the approved patch as a **virtual milestone** (stored in `gov:` prefix,
undo-safe). Rules read milestones through one accessor, so nothing else changes. Relay and delegate nodes need no restart;
`GET /api/node/configuration` and the metrics page show the pending and applied proposals.

### Safety

- Only whitelisted keys, bounded ranges, one active proposal per key at a time.
- `minCoreVersion` proposals require the proposer's own node to run that version (signed announce, SHIP-8).
- Emergency stop: 15 of 21 `ProposalCancel`-votes (kind 2, `approve = 0`) within the window cancels any proposal.

## Rationale

11-of-21 is a simple majority of the elected body; two weeks is long enough for holders to react (votes count immediately,
delegate ranks change per round) and short enough to be useful. Parameter-only scope avoids "governance can change
anything" risks; code changes still follow SHIP-1.

## Backwards Compatibility

New typeGroup; Rust-only network. The whitelist can only grow through a new SHIP.

## Reference Implementation

Not started. Touch points: `models` (typeGroup 4), `rules`, `storage` (`gov:`), `config::Network::milestone` (virtual
milestones), `api/node.rs`, metrics page tab **governance**, `sth-cli tx proposal | proposal-vote`.
