```
  SHIP: 37
  Title: Netfory Web4 Integration - ntfryData and sth:// Content Links
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Interface
  Created: 2026-06-12
  Last Update: 2026-09-03
```

## Abstract

Defines how on-chain objects point to off-chain content in the **Netfory** Web4 network: the `ntfryData` field of sObjects,
tokens and NFTs carries an `sth://` URI or a content hash; Netfory nodes (running on the same Iroh identities as SmartHoldem
nodes, [SHIP-4.md](SHIP-4.md)) store and serve the content peer-to-peer with **iroh-blobs** (BLAKE3-verified, resumable transfer), so
manifests, documents, media and contract programs are available without any web server, and their integrity is anchored
by the chain.

## Motivation

Chains store facts; documents, images and applications live elsewhere - usually on centralised hosting that disappears or
changes. SmartHoldem already runs on Iroh; Netfory extends the same transport to content so that "the wallet shows a token
logo / an NFT / a company profile" never depends on a URL that someone controls.

## Specification

### URI scheme

```
sth://<hash>[/<path>]                  content-addressed: BLAKE3 hash (base32, 52 chars) of a blob or collection
sth://<name>.<type>[/<path>]           name-addressed: resolved through the sObject name index (SHIP-13) -> its ntfryData -> hash
```
Examples: `sth://coffee.5/manifest.json` (token COFFEE manifest), `sth://k5…q4a/image.png`.

### Resolution

1. Name -> sObject (`GET /api/sobj/search {name, type}`) -> `ntfryData` (must itself be a hash URI) - one indirection, so the
   owner can update content without touching the chain-anchored name.
2. Hash -> Netfory: `iroh-blobs` fetch from any provider announcing the hash on the `ntfry-content` gossip topic or via the
   content DHT; verification is inherent (hash of received bytes).
3. Nodes may act as **pinning providers** for hashes referenced by chain objects they index (tokens with manifests, NFTs,
   Safe Contract programs) - `node.yaml -> ntfry.pin: chain-referenced | all | none`.

### Documents

- **Token manifest** ([SHIP-15.md](SHIP-15.md)) may point to an extended manifest (`sth://…/manifest.json`: long description, links,
  audit reports) beyond the 8 KB on-chain limit.
- **NFT items** ([SHIP-26.md](SHIP-26.md)): `sth://…/item.json` with `image`, `attributes`, `license`.
- **Governance proposals** ([SHIP-23.md](SHIP-23.md)): rationale documents.
- **Sealed documents** ([SHIP-34.md](SHIP-34.md)): encrypted blobs; the chain holds only the hash.
- **Oracle feeds** ([SHIP-33.md](SHIP-33.md) `oracleValue`): signed JSON documents published by named sObjects (type 7 `oracle`), verified by
  the publisher's key.

### Gateways

`GET /ntfry/<hash-or-name>[/<path>]` on any node exposes content over HTTP for legacy browsers; the response carries
`X-Sth-Hash` so clients can verify. Wallets and explorers embed content via `sth://` natively.

### Chain anchoring

Only hashes/names are on chain; content changes require a new hash -> a new `update` transaction (visible history of every
change of a manifest or document).

## Rationale

Content addressing via the same P2P stack removes the last centralised piece (asset hosting) from the user experience; a
single indirection through names keeps human-readable links while retaining verifiability.

## Backwards Compatibility

`ntfryData` is already free-form ([SHIP-13.md](SHIP-13.md)); the scheme is a convention wallets adopt progressively.

## Reference Implementation

Not started in `sth-core` (gateway, pinning); `ntfryData` plumbing exists.
