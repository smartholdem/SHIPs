```
  SHIP: 9
  Title: Isolated Networks from the Command Line (init newnet)
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Tooling
  Created: 2025-12-08
  Last Update: 2026-06-19
```

## Abstract

`sth-core init newnet` generates a complete, isolated SmartHoldem network in one command: genesis block, delegate keys,
`network.json`, `milestones.json`, `exceptions.json`, `node.yaml` and a treasury wallet. The generated network runs the
same binary and the same rules as mainnet, with feature milestones (`tokens`, `sobjV2`, `pq`) placed at chosen heights.

## Motivation

Every consensus feature must be exercised end-to-end by several nodes before a mainnet milestone is scheduled. Building a
test network by hand (genesis, nethash, delegate registration, config files) took hours and was error-prone; isolation
between test networks and mainnet must be guaranteed by construction.

## Specification

```
sth-core init newnet --out DIR --delegates N [--seed S] [--ticker TST] [--title TestNet]
                     [--tokens-at H] [--pq-at H] [--treasury AMOUNT] [--delegate-stake AMOUNT]
                     [--api-port 4004] [--legacy-port 4002]
```

- **Determinism**: with `--seed` every key, address and the genesis block are reproducible; without it, random.
- **Genesis**: transfers the treasury supply, registers N delegates and casts their self-votes; block 1 is signed by
  delegate 1; `nethash = sha256(genesis bytes)`, `genesisBlockId` patched into `network.json` like mainnet.
- **Milestones**: mainnet rules folded into one milestone at height 1 (v2 wire format, Schnorr blocks, sObjects, HTLC,
  8-second slots, `activeDelegates = N`), plus optional `{ height: H, tokens: true }` and
  `{ height: H, pq: { active: true, feePerByte: 10000, commitmentGrace: 86400 } }`; `sobjV2: true` and `strictBalance: true`
  from the start.
- **Isolation**: distinct `nethash`, address prefix (`pubKeyHash`), gossip topic ids (derived from nethash, [SHIP-4.md](SHIP-4.md)) and
  legacy protocol version; a test node cannot connect to or accept mainnet blocks.
- **Second node**: copy the directory, change ports, point `p2p.bootstrap` at the first node's iroh id - no seed servers.
- Output: `delegates.json` (usernames, passphrases, addresses), `node.yaml` with `network_dir: .`, and a summary
  (nethash, treasury address, first delegate).

## Rationale

Generating the network with the production code path (not a fixture) means a passing test network is real evidence for
a rollout. Determinism makes CI reproducible.

## Reference Implementation

`src/newnet.rs`, `src/main.rs` (`init newnet`), test `tests/newnet.rs`.
