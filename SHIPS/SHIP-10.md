```
  SHIP: 10
  Title: Standalone Client sth-cli
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Active
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Tooling
  Created: 2025-12-22
  Last Update: 2026-06-19
```

## Abstract

`sth-cli` is a standalone command-line client (comparable to `bitcoin-cli`) that talks to any node over the REST API:
wallet creation (BIP-39), read commands, transaction building and signing for every transaction type, and a live `watch`
feed. It shares the transaction builder with the node (`sth-core tx`) so both always produce identical bytes.

## Motivation

Operators and integrators need a scriptable, JSON-capable client that does not require a local database, and a reference
signer for new transaction types (sObjects, tokens, v3) before wallets support them.

## Specification

```
sth-cli [--api URL] [--json] <command>
  create-wallet [--words 12|24]           BIP-39 mnemonic, address, public key (offline)
  status | wallet ADDR | token TICKER | obj ID|NAME [--type N] | market [--type N] | tx ID [--wait]
  pq-commitment --second-passphrase P     Quantum Shield stage A commitment (SHIP-18)
  watch ADDR [--every S] [--tail N]       live feed: STH, tokens, sObjects, orders, shield events
  tx <subcommand>                         build, sign, broadcast (STH_PASSPHRASE / --passphrase, --nonce, --second-passphrase)
     transfer | vote | unvote | delegate-register | second-signature
     obj-register | obj-update | obj-resign | obj-transfer | obj-sell | obj-buy        (SHIP-13/16)
     token-init | token-transfer | token-mint | token-burn | token-meta               (SHIP-14/15)
     pq-register [--old-second-passphrase P]                                          (SHIP-20)
```

- Chain parameters (fees, `pubKeyHash`, milestone flags, `pq.feePerByte`) are read from `/api/node/configuration`; the
  client never hard-codes consensus constants.
- Second signatures: `--second-passphrase` (or `STH_SECOND_PASSPHRASE`) produces a legacy v2 second signature for
  wallets with `secondPublicKey`, and automatically a **v3** transaction with an ML-DSA-44 block plus the fee surcharge
  for PQ-locked wallets ([SHIP-19.md](SHIP-19.md), [SHIP-20.md](SHIP-20.md)).
- Nonce handling: wallet nonce + 1 with one automatic retry on `ERR_NONCE`; `--json` prints machine-readable output.
- `watch` polls the wallet's transactions and prints human descriptions ("sObject registered", "sale order", "token
  received", "Quantum Shield registered").

## Rationale

A separate binary keeps the node free of interactive concerns and gives integrators a tool with no consensus state;
sharing
`src/cli.rs` with `sth-core tx` guarantees byte-identical transactions.

## Reference Implementation

`src/bin/sth-cli.rs`, `src/cli.rs`, `src/crypto/mnemonic.rs`; docs `docs/CLI.md`.
