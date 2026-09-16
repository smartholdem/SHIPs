```
  SHIP: 33
  Title: Safe Contracts - Declarative, Non-Turing-Complete sObject Programs
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-03
  Last Update: 2026-09-01
```

## Abstract

SmartHoldem's own contract model that **cannot** have reentrancy, gas exhaustion, integer overflow or unbounded loops,
because it has no loops, no dynamic dispatch and no arbitrary state: a **Safe Contract** is a sObject ([SHIP-13.md](SHIP-13.md), `type 6`)
holding a small declarative program built from audited **primitives** (conditions and actions) that the node evaluates in
bounded time. Contracts are configured, not coded - like HTLC, multisignature and the sObject market, which are the first
three primitives already in consensus.

## Motivation

Most losses on contract platforms come from bespoke code re-implementing the same few patterns (escrow, vesting, splits,
auctions, subscriptions). SmartHoldem already runs those patterns as native rules with zero contract exploits. Safe
Contracts generalise this: users compose primitives; the network implements each once, with tests and formal bounds.

## Specification

### Program

```json
{ "v": 1,
  "params":  { "beneficiary": "S…", "release": 12000000, "share": 30 },
  "rules": [
    { "when": { "all": [ { "heightAtLeast": "$release" }, { "callerIs": "$beneficiary" } ] },
      "then": [ { "paySth": { "to": "$beneficiary", "percentOfBalance": 100 } } ] }
  ] }
```

Stored as `ntfryData`-sized JSON in the sObject (≤ 4 KB, `type 6`, `subType` = template id); the contract's funds live in a
**contract address** derived from the registration id (`RIPEMD160(sha256("sth-contract" ‖ id))`, no private key).

### Primitives (v1)

Conditions: `heightAtLeast/Before`, `timestampAtLeast`, `callerIs`, `callerHasToken`, `preimage(hash)`, `signaturesAtLeast(k, keys)`
(threshold approvals via `ContractApprove`), `balanceAtLeast`, `oracleValue` ([SHIP-37.md](SHIP-37.md) signed Netfory feed).
Actions: `paySth`, `payToken`, `splitSth(shares)`, `lockHtlc`, `transferSobj`, `setParam` (bounded, only declared params),
`emit(event)`. Every primitive has a fixed cost; a program has ≤ 16 rules x ≤ 8 conditions x ≤ 8 actions.

### Execution model

`ContractCall` (`typeGroup 6`, `type 1`, `contractId ‖ ruleIndex ‖ argsLen ‖ args`) evaluates **one** rule: all conditions
must hold, then actions run in order, atomically, in the caller's block. No rule can call another contract; actions cannot
trigger rules (no reentrancy by construction). Evaluation is O(rule size) - no gas needed, a flat fee per action.
Deployment: `ContractDeploy` (`type 0`) registers the sObject and validates the program against the primitive schema;
`ContractApprove` (`type 2`) records a signer's approval for threshold rules.

### Templates

Templates are audited programs with parameters: `escrow`, `vesting`, `revenue-split`, `dutch-auction`, `subscription`,
`dead-man-switch` ([SHIP-34.md](SHIP-34.md)). Wallets show a template by name and its parameters - users never see a program they did not
choose.

### Guarantees

Termination and cost are bounded statically; funds can only move by declared actions; integer arithmetic is checked
(overflow = rule fails, state untouched); upgrades are impossible (deploy a new contract) - what you read is what will run.

## Rationale

The set of primitives grows only through SHIPs with tests; expressiveness is traded for the property that a contract can
be fully understood by reading its parameters. This matches the network's DNA (native tokens instead of token contracts).

## Backwards Compatibility

New typeGroup; [Rust-only network](https://github.com/smartholdem/sth-core-rust).

## Reference Implementation

Not started.
