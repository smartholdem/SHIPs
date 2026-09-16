```
  SHIP: 1
  Title: SHIP Purpose and Guidelines
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Process
  Created: 2020-09-01
  Last Update: 2026-06-19
```

## What is a SHIP?

A SmartHoldem Improvement Proposal (SHIP) is a design document providing information to the SmartHoldem community, or
describing a new feature for the network, its node software (`sth-core`), tooling or processes. The SHIP should provide a
concise technical specification of the feature and a rationale for it. The author is responsible for building consensus
within the community and documenting dissenting opinions.

SHIPs are the primary mechanism for proposing new features, collecting community input and documenting the design
decisions that have gone into SmartHoldem. Because SHIPs are kept as text files in a versioned repository, their revision
history is the historical record of the feature proposal.

## SHIP Types

- **Standards Track** - any change that affects most or all implementations: consensus rules, transaction types, wire
  formats, networking protocols, REST API, or any change to the interoperability of applications using SmartHoldem.
- **Informational** - design issues, general guidelines or information; does not propose a new feature.
- **Process** - a process surrounding SmartHoldem, or a change to a process (like this document).

Standards Track SHIPs carry a **Category**: `Core`, `Networking`, `Interface`, `Tooling`.

## Workflow

1. Discuss the idea in <https://github.com/smartholdem/SHIPs/issues>.
2. Open a pull request adding `SHIPS/ship-N.md` with status **Draft** (the editors assign the number).
3. Once the reference implementation is merged into `sth-core` and covered by tests the SHIP becomes **Accepted**
   (if activation waits for a milestone or a coordinated delegate upgrade) or **Active** (rules enforced on mainnet).
4. A SHIP that must never change again (wire formats, hashing) is marked **Final**; further changes need a new SHIP.
5. Superseded or abandoned proposals are marked **Replaced** / **Withdrawn**.

Consensus-affecting SHIPs activate through a **milestone** in `milestones.json` (height-gated flag). A milestone may only be
scheduled when every active delegate (all 21) and the reserve run a core version enforcing the new rules; readiness is
verified on the Delegate Dashboard (SHIP-8) and the rollout follows the checklist in `docs/MAINNET-ROLLOUT-PQ_RU.md`.

## Format

Every SHIP starts with the preamble shown above, followed by:

- **Abstract** - a short (~200 word) description of the technical issue being addressed.
- **Motivation** - why the existing protocol is inadequate.
- **Specification** - the technical specification, detailed enough for interoperable implementations.
- **Rationale** - design decisions and alternatives considered.
- **Benefits** / **Backwards Compatibility** / **Security Considerations** - as applicable.
- **Reference Implementation** - module paths and tests in `sth-core`.

## Editors

Current editor: TechnoLog. Editors do not judge proposals on merit; they check completeness, format and assign numbers.
