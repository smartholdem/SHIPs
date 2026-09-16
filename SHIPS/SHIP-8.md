```
  SHIP: 8
  Title: Operator Metrics Page and Delegate Dashboard (Version Guard)
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Interface
  Created: 2025-11-24
  Last Update: 2026-06-19
```

## Abstract

Every `sth-core` node serves a self-contained HTML metrics page at `GET /` (and on an optional dedicated metrics port)
backed by two JSON endpoints, `GET /api/ntfry/metrics` and `GET /api/ntfry/delegates`. The page shows chain, peer, mempool,
token, market and Quantum Shield statistics and a **Delegate Dashboard** listing the 21 active delegates with the software
they run, so that consensus upgrades can be scheduled with evidence rather than guesses.

## Motivation

Milestones that legacy nodes cannot validate may only be activated when all active delegates run a sufficiently new core
(SHIP-1). Operators need a trustworthy, server-less way to see who is ready.

## Specification

### Delegate Dashboard

For each of the top‑21 delegates (by the vote index, [SHIP-7.md](SHIP-7.md)):

| field | source |
|---|---|
| `implementation` | `rust` - proven by a signed `Delegates` gossip announce ([SHIP-4.md](SHIP-4.md)); `legacy` - inferred from legacy-only block evidence ([SHIP-5.md](SHIP-5.md)); else `unknown` |
| `version` | core version from the announce |
| `outdated` | `version < milestone.minCoreVersion` of the current height |
| `node`, `lastSeen` | iroh node id (short) and seconds since the announce |

**Version Guard**: the milestone field `minCoreVersion` (string, semver) marks delegates below it with a warning
(`⚠ update to ≥ vX.Y.Z`) and counts them in `outdated`. The check is informational - the node never refuses blocks based on
announced versions - but it is the gating input of every rollout checklist.

### Metrics JSON

`node` (version, height, uptime, `pqStage`, `pqCommitments`, `pqActive`), `peers` (iroh / legacy / gateways),
`mempool` (`count`, `max`, `bytes`, `maxBytes`), `tokens` (count, manifests, logos), `market` (open orders with token
manifests), `delegates` (dashboard above), `sync` (behind, batch rate).

### Page

Static HTML + vanilla JS, no external assets, refresh every 5 s, tabs: overview, delegates, tokens, market. Interactive
elements carry `data-testid` attributes for automated checks.

## Rationale

Signed announces make `rust` a proof, not a claim; legacy inference is explicitly marked as such. Keeping the page inside
the binary avoids a separate monitoring stack for small operators.

## Reference Implementation

`src/api/ntfry.rs`, `src/api/metrics.html`, `src/p2p_iroh/delegates.rs`; test `tests/api.rs`.
