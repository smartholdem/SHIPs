```
  SHIP: 42
  Title: Service Nodes (Masternodes) Protocol & Netfory Infrastructure Incentives
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core / Interface / Protocol
  Created: 2026-09-17
  Last Update: 2026-09-17
  Requires: 13, 14, 41
```

## Abstract

This proposal defines a **Service Node** (masternode) layer for SmartHoldem: a class of non-forging nodes that lock a
collateral of STH, run 24/7 and provide verifiable network services - Iroh relays, Netfory content seeding, messenger
relaying, `u://` microblog relaying and `api://` Web2->Web4 provider gateways - in exchange for protocol-level rewards and
peer-to-peer micro-payments.

Service Nodes **do not forge blocks**. Block production remains the exclusive right of the Top-21 DPoS delegates
([SHIP-1.md](SHIP-1.md), [SHIP-11.md](SHIP-11.md), [SHIP-35.md](SHIP-35.md)). Service Nodes are registered on-chain as SmartObjects ([SHIP-13.md](SHIP-13.md)) of a dedicated type, their
health is attested by the active delegates over Iroh gossip (*Proof of Service*), and a configurable share of the dynamic
transaction fees ([SHIP-41.md](SHIP-41.md)) and/or a dedicated block reward is paid out to nodes that met the uptime threshold of the
previous payout epoch.

The proposal is adaptation needed to the PQ node (`sth-core`), the Iroh transport
([SHIP-4.md](SHIP-4.md)), the sObject state machine ([SHIP-13.md](SHIP-13.md), [SHIP-16.md](SHIP-16.md)) and native tokens ([SHIP-14.md](SHIP-14.md)).

## Motivation

1. **The altruistic node problem.** Relay and API nodes (`sth-core run` without `delegate.secrets`) pay for bandwidth,
   disk and a public IP but receive nothing from the protocol. Today every wallet, explorer and Netfory application depends
   on a volunteer gateways (`legacy_listen`, `api.cors_origins`, `netfory-provider`). As Web4 traffic grows -
   static sites, messenger buffering, microblog feeds, API tunnels - the cost is borne by a shrinking set of operators.
2. **Connectivity for the Netfory / Web4 ecosystem.** Iroh hole-punching works only as well as the relay coverage
   (`p2p.iroh.relays`, [SHIP-4.md](SHIP-4.md)). Home delegates behind NAT, mobile wallets and browsers need geographically distributed,
   high-bandwidth relays that no delegate is obliged to run.
3. **Tokenomics.** Locking 10 000 STH per Service Node removes liquidity from the market in exchange for a predictable
   yield tied to *useful work*, complementing the vote-weight lock-up of DPoS.
4. **Separation of powers.** Keeping consensus with 21 delegates while distributing *infrastructure* over hundreds of
   Service Nodes gives decentralisation where it matters (data availability, censorship resistance) without touching block
   finality (SHIP-35).

## Specification

### Terms

| term | meaning |
|---|---|
| **Service Node (SN)** | A `sth-core` node with a registered `masternode` sObject whose collateral is locked and whose Proof of Service is current. |
| **Collateral** | `masternodes.collateral_amount` STH held by the *collateral address* - the wallet that registered the sObject. |
| **Service role** | One of `relay`, `seeder`, `messenger`, `microblog`, `provider` (§ Service Types). |
| **Attestor** | An active delegate of the current round (Top-21) that publishes health attestations over Iroh gossip. |
| **Payout epoch** | `masternodes.payout_epoch_rounds` consecutive DPoS rounds (default 100 rounds ≈ 4.7 h at 21 × 8 s). |
| **Reward Pool** | The virtual account accumulating the Service Node share of fees/rewards during an epoch (§ Rewards). |
| **PoServ** | Proof of Service - the on-chain-verifiable uptime score derived from attestations. |

### Collateral and Registration

Registration is a SHIP-13 sObject transaction (`typeGroup 2`, `type 6`) with a new **reserved sObject type**:

```json
{ "type": 7, "subType": 0, "action": 0,
  "data": {
    "name": "sn-fsn7",
    "ntfryData": "sth://sn/<iroh EndpointId>?roles=relay,seeder,provider&api=https://node-pq01.smartholdem.io"
  } }
```

* `type = 7` - Service Node object (`SOBJ_TYPE_SERVICE_NODE`). Registration MUST fail unless the milestone flag
  `masternodes.enabled` is true at the block height.
* `name` - unique network-wide (SHIP-13 rules), used as the operator-visible label in `/api/masternodes`.
* `ntfryData` - `sth://sn/<EndpointId>` where `<EndpointId>` is the node's Iroh public key (`sth-core iroh-id`). Query
  parameters: `roles` (comma-separated subset of § Service Types, required), `api` (optional public REST URL), `storage`
  (optional advertised seeding capacity in GiB). Total length ≤ 255 bytes (sObjV2).
* The sObject is **never transferable** (`action 3/4/5` rejected, as for `type 4` delegate objects).
* `action 1` (update) replaces `ntfryData` - roles, endpoint or URL may change; the EndpointId change re-starts the
  PoServ history (§ Proof of Service). `action 2` (resign) frees the collateral requirement immediately.

**Collateral rule.** At every block the node evaluates, for every registered Service Node object,

```
active(sn) := !sn.resigned
           && ownerWallet(sn).balance ≥ collateral_amount(height)
           && ownerWallet(sn).balance - Σ(outgoing amounts+fees in this block) ≥ collateral_amount(height)
```

The collateral is **not** moved to a separate lock address; it is a *balance floor* on the collateral address, exactly as
delegate vote weight is a live balance (SHIP-1). Spending below the floor sets `active = false` at that height: the node
stops accruing PoServ and is excluded from the next payout. Topping the balance back up re-activates it at the next block -
but the uptime window restarts (§ Proof of Service), so short-term shuffling of the collateral costs the operator a full
epoch of rewards.

A wallet MAY register at most **one** Service Node object (`SnAlreadyRegisteredError`). One Iroh EndpointId MAY back at
most one Service Node object (`EndpointAlreadyRegisteredError`, checked case-insensitively on the base32 id).

Fees: registration uses the SHIP-13 `register` fee (50 STH) and is *not* subject to dynamic fees (fixed per action,
SHIP-41 § Core-only scope). All milestone parameters:

```json
{
  "height": 12500000,
  "masternodes": {
    "enabled": true,
    "collateral_amount": "1000000000000",
    "fee_share_percent": 30,
    "block_reward": "0",
    "payout_epoch_rounds": 100,
    "min_uptime_percent": 90,
    "min_attestors": 11,
    "max_payout_recipients": 256
  }
}
```

All amounts are strings in smartoshi; `collateral_amount` defaults to `1000000000000` (10 000 STH). Every value is
milestone-scoped and therefore changeable by a coordinated milestone update (see `docs/MILESTONES_RU.md`).

### Service Types

Roles are advertised in `ntfryData.roles` and verified by attestors role-by-role. A node MAY provide several roles; a
role that fails verification lowers only that role's score.

| role | service | verification probe (attestor -> SN) | notes |
|---|---|---|---|
| `relay` | **N1 Network Relay** - Iroh relay + hole-punching server (`relay: true` in `p2p.iroh`), high bandwidth, public IP or `relays: [https://…]` endpoint | Iroh RPC `SnProbe { role: relay }` + a 64 KiB echo over a *relayed* path through the SN's relay URL; latency and throughput recorded | Home delegates (SHIP-4 §1) select these relays first. |
| `seeder` | **Content Seeder** - dedicated local storage (`allocated_storage_gb`) seeding Web4 static sites, files and Netfory network state addressed by `sth://` pointers (SHIP-37) | `SnProbe { role: seeder, want: <random chunk id from the seeding manifest published by the SN> }` - the SN must return the chunk and its BLAKE3 hash within 2 s | Optional per node; capacity advertised in `storage=` GiB. |
| `messenger` | **Messenger Relay** - low-latency routing and encrypted store-and-forward buffering for Netfory Messenger (SHIP-40 E2EE envelopes) | attestor stores a 1 KiB envelope for a random recipient key, retrieves it from a second attestor session ≤ 30 s later | Buffer TTL ≥ 72 h, ≥ 64 MiB per recipient. |
| `microblog` | **Microblog Relay / Seeder** - indexing, caching and relaying `u://` feeds (`u://<publicKey>/<slug>`) and social graph updates | `SnProbe { role: microblog, feed: <random known feed> }` - the SN returns the feed head signed by the author, no older than the attestor's own copy minus 2 slots | Feed heads are signed by publishers; the SN never rewrites content. |
| `provider` | **Netfory Provider** - reverse-proxy of Web2 API services into the P2P network under `api://<sn-name>/<service>` (game server APIs, RPC endpoints, REST) | `SnProbe { role: provider, path: "/_health" }` through the Iroh RPC channel; attestor checks the signed provider manifest (list of exposed `api://` services, pricing) | Premium relaying is paid with micro-payments (§ Service Micro-Payments). |

Probe formats are Iroh RPC messages on the existing `sth/<nethash>/rpc` ALPN (SHIP-4): `SnProbe` / `SnProbeReply` with a
32-byte attestor nonce echoed and signed by the SN's *Iroh* key. A reply is valid only if signed by the EndpointId
registered in `ntfryData`.

### Proof of Service (PoServ)

PoServ turns attestor observations into an on-chain-verifiable uptime score without adding a transaction per probe.

1. **Probing.** Every active delegate probes every active SN once per `probe_interval` (default 1 round). Probes are
   spread deterministically over the round: SN `k` is probed by delegate `d` at slot
   `(hash(nethash ‖ round ‖ endpointId) + d.rank) mod 21` so that no SN is hit by 21 probes in the same slot.
2. **Attestations.** After each probe the delegate publishes on the Iroh gossip topic
   `sha256("sth/<nethash>/service")` the message

   ```
   SnAttestation { round, endpointId, roles_ok: bitmask, latency_ms, publicKey, signature }
   signature = Schnorr(sha256("sth-sn-attest-v1" ‖ round LE ‖ endpointId ‖ roles_ok ‖ latency_ms))
   ```

   over the delegate's forging key - the same key that signs blocks and SHIP-35 finality votes, so attestations cannot be
   forged by non-delegates and equivocating attestations (two different `roles_ok` for the same `(round, endpointId)`) are
   slashable exactly like double votes (SHIP-35 § Equivocation: 30-round forging ban).
3. **Aggregation.** Every node keeps a sliding table `sn_uptime[endpointId][round] = number of distinct attestors with
   roles_ok ≠ 0`. A round counts as **up** when `attestors ≥ min_attestors` (default 11 of 21 - a majority of the active
   set). Uptime of an epoch = up-rounds / epoch rounds.
4. **Commitment.** The forger of the **first block of a payout epoch** embeds in that block's `payloadHash` preimage an
   extra leaf: `snRoot = merkleRoot(sorted (endpointId, upRounds, rolesMask))` for the epoch that just ended. Because
   `payloadHash` is already signed (SHIP-11), `snRoot` becomes part of the chain; verifying nodes recompute it from their
   own attestation table and reject the block when it differs (`SnRootMismatchError`). Nodes that were offline for part of
   the epoch fetch the missing attestations via Iroh RPC `GetSnAttestations { round_from, round_to }` from ≥ 3 peers (same
   pattern as `GetFinality`, SHIP-35).
5. **Score.** `score(sn) = upRounds / payout_epoch_rounds`. An SN qualifies for payout when
   `score ≥ min_uptime_percent / 100` **and** it was `active` (collateral) in **every** round of the epoch.

An SN whose EndpointId or roles changed inside the epoch (`action 1`) is treated as newly registered from that round: its
`upRounds` before the change are discarded.

### Rewards Distribution

**Sources.** The Reward Pool of an epoch receives, per block:

* `fee_share_percent` % of the block's `totalFee` - the dynamic fees of SHIP-41 (core typeGroup 1) plus fixed sObject/token
  fees. The forging delegate receives the remaining `100 − fee_share_percent` %.
* `block_reward` smartoshi of dedicated Service Node reward (default `0` on mainnet, where block rewards are zero; a
  testnet or a future milestone may fund the pool from emission - `Network::supply()` accounts for it).

`fee_share_percent = 0` and `block_reward = "0"` make the proposal a pure registry with no economic effect - the
recommended first activation stage (see § Backwards Compatibility).

**Accounting.** The pool is a protocol account with no private key: `wallet("SN_POOL")` derived from
`sha256("sth-sn-pool" ‖ nethash)` in the network's address version. Its balance changes only by the rules above and by
payouts; a transaction *from* it is invalid (`ProtocolAccountError`).

**Payout.** At the first block of epoch `E+1` (the block that commits `snRoot(E)`), the node applies a *virtual
multi-payment* (no on-chain transaction, no fee): the pool balance is split **equally** among qualified SNs, with a
+10 % weight for each additional verified role above the first (`weight = 1 + 0.1 × (roles − 1)`, capped at 1.5) and paid
to each SN's **reward address**. Payouts are recorded as synthetic entries in the block's `transactions` view
(`GET /api/blocks/:id/transactions?includeProtocol=true`) and as a `snPayout` field in `GET /api/blocks/:id` so that
explorers can show them; they are not counted in `numberOfTransactions`. If more than `max_payout_recipients` qualify, the
lowest-scoring nodes roll over to the next epoch with their score carried (round-robin by `endpointId` hash on ties). If no
SN qualifies the pool carries over.

**Reward address.** The currently unused `rewards.reward_address` in `node.yaml` becomes the SN's payout destination
(defaults to the collateral address when empty). It is advertised in `ntfryData` as `reward=<address>`, so the on-chain
object - not the node configuration - is authoritative.

### Service Micro-Payments

Rewards cover *availability*; heavy *usage* (paid bandwidth, premium `api://` relaying, priority messenger buffering) is
settled directly between client and SN:

* The SN publishes a signed **price manifest** in its provider/seeder metadata (`price_per_mib`, `price_per_call`, token
  `STH` or a SHIP-14 native token id).
* The client pays with an ordinary transfer or token transfer (SHIP-14 `TokenTransfer`) carrying `vendorField =
  "sn:<endpointId>:<sessionId>"`; the SN unlocks the session once the transaction is in the pool (`recentlySeen`) or in a
  block, according to its own risk policy. Dynamic fees (SHIP-41) make sub-cent payments practical (≈ 0.008 STH).
* Streaming sessions MAY use SHIP-9 HTLC locks (`htlcLock` -> periodic `htlcClaim` by the SN) for pay-as-you-go without
  per-request transactions.

Micro-payments are outside consensus; this section fixes only the `vendorField` convention so that explorers and
wallets can attribute the traffic.

### Configuration

`node.yaml` gains a `services` block (alias `masternode` accepted):

```yaml
rewards:
  reward_address: "S…"            # payout destination (defaults to the collateral address)
services:
  enabled: true                   # run the SN probe responder + advertised roles
  collateral_secret_file: ""      # optional: file with the collateral wallet passphrase, only for `sth-cli sn register`
  roles: [relay, seeder, provider]
  allocated_storage_gb: 50        # seeder: local capacity under db_path/sn-store
  provider:
    manifest: ./provider.yaml     # api:// services, upstream URLs, prices
  messenger:
    buffer_mb: 512
p2p:
  iroh:
    relay: true                   # required for the relay role
```

`sth-cli` additions: `sth-cli sn register --name --roles --api --storage`, `sth-cli sn update`, `sth-cli sn resign`,
`sth-cli sn status` (score, next payout, missed rounds by attestor). `sth-cli delegate-setup` is unchanged.

### API

| method | path                                       | description                                                                                                                                                                                                                                                             |
|--------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GET    | `/api/masternodes`                         | active and inactive Service Nodes: `name`, `address`, `rewardAddress`, `publicKey`, `endpointId`, `roles`, `active`, `collateral`, `uptime` (current epoch, %), `lastSeen`, `latencyMs`, `api`, `storageGb`; filters `role=`, `active=`, pagination as `/api/delegates` |
| GET    | `/api/masternodes/:id`                     | by name, address or EndpointId; includes `history` (last 10 epochs: score, paid)                                                                                                                                                                                        |
| GET    | `/api/masternodes/:id/attestations?round=` | attestations of one round (who saw it, roles bitmask, latency)                                                                                                                                                                                                          |
| GET    | `/api/masternodes/pool`                    | pool balance, current epoch progress, projected per-node payout                                                                                                                                                                                                         |
| GET    | `/api/ntfry/metrics`                       | extended with `serviceNodes: { total, active, qualified, poolBalance, epochRound, myScore }`                                                                                                                                                                            |
| GET    | `/api/node/configuration`                  | `constants.masternodes` mirrors the milestone block                                                                                                                                                                                                                     |

Iroh RPC (SHIP-4 ALPN): `SnProbe`, `SnProbeReply`, `GetSnAttestations`, `GetSnRoot { epoch }`. Gossip topic
`sha256("sth/<nethash>/service")`.

Storage ([SHIP-3.md](SHIP-3.md) prefixes): `sn:` (endpointId -> object id, roles, since), `sa:<round>:<endpointId>` (attestations, pruned
after 2 epochs), `sr:<epoch>` (committed `snRoot` + payout list).

## Rationale

* **sObject instead of a new transaction type.** SHIP-13 already gives unique names, ownership, updates and resignations
  with wire format, fees, indexes and CLI. A reserved `type = 7` reuses all of it; only the collateral floor and the
  non-transferability rule are new - both already exist for `type = 4` delegate objects.
* **Balance floor instead of a lock address.** A lock address would need unlock transactions, time-locks and a new
  balance-affecting rule in `accumulate_deltas`. A floor is one comparison per block and mirrors how vote weight works;
  operators can always exit instantly by spending.
* **Delegates as attestors.** They are the only set with keys already bound to consensus and slashing (SHIP-35). Using a
  majority (`min_attestors = 11`) means a single hostile or misconfigured delegate cannot deny or grant rewards.
* **`snRoot` in `payloadHash` rather than a payout transaction.** Legacy nodes hash `payloadHash` opaquely, so the commit
  costs zero bytes on the wire and needs no new transaction type; payouts as *virtual* balance changes keep blocks small
  (256 recipients would otherwise be a 10 KiB multi-payment every epoch).
* **Equal split with a mild multi-role bonus** - not proportional to collateral - so that running *more* useful services
  pays more than parking *more* coins, and so that Sybil splitting (§ Security) is never more profitable than one honest
  node.
* **Micro-payments off-consensus.** Usage-based billing varies per service; fixing only the `vendorField` convention
  keeps the protocol small while making traffic attributable.

## Backwards Compatibility

* Nothing changes for nodes and wallets until the milestone `masternodes.enabled` is set. Registration transactions before
  that height are rejected with `SobjTypeNotActiveError`; the sObject type `7` is reserved from this SHIP on so that
  application developers do not use it.
* Legacy (Node.js) nodes cannot verify `snRoot` or apply virtual payouts. Therefore **stage 1** (registry only:
  `fee_share_percent = 0`, `block_reward = "0"`) is safe while legacy nodes forge; wallets see the objects, explorers list
  `/api/masternodes`, PoServ runs and is visible, nothing is paid. **Stage 2** (payouts) requires the delegate set to be
  fully on `sth-core`, like SHIP-35 hard finality and SHIP-14 tokens.
* `rewards.reward_address` keeps its current semantics (unused) until stage 2; the config comment is updated to point to
  this SHIP.
* `/api/blocks/:id` keeps its shape; `snPayout` is an additive field, protocol entries appear only with
  `includeProtocol=true`.

## Reference Implementation

Planned in `sth-core-rust`: `models::sobj::TYPE_SERVICE_NODE`, `storage` prefixes `sn:`/`sa:`/`sr:`, `p2p_iroh::service`
(probes, attestations, `GetSnAttestations`), `delegate::block_builder` (`snRoot` leaf), `api::masternodes`,
`sth-cli sn *`. Test suites: `tests/service_nodes.rs` (registration + collateral floor), `tests/poserv.rs` (attestation
aggregation, snRoot determinism, payout split), `tests/newnet.rs` (milestone activation on an isolated testnet).

## Security Considerations

* **Spam / cheap registration.** 50 STH registration fee (SHIP-13) plus the 10 000 STH floor per object; objects without
  collateral are inactive and never probed, so they cost attestors nothing.
* **Sybil resistance.** Rewards are per *qualified node*, so splitting 100 000 STH into ten nodes yields ten equal shares -
  but each node must independently answer probes with real bandwidth/storage; the multi-role bonus is capped at 1.5× and the
  epoch payout is bounded by the pool, so the marginal Sybil node dilutes the attacker's own nodes as much as everyone
  else's. `max_same_subnet` (default 4 SNs per /24, checked by attestors on the probed address) limits data-centre farming.
* **Fake uptime.** Attestations are signed by delegate forging keys; a non-delegate cannot forge them and a delegate that
  attests inconsistently (equivocation) is banned from forging for 30 rounds ([SHIP-35.md](SHIP-35.md)). `min_attestors = 11` requires a
  majority of the active set to be compromised to grant undeserved rewards. Probe nonces (32 random bytes, signed by the
  SN's Iroh key) prevent replay; seeder/microblog probes fetch *random* content so a node cannot pre-compute answers.
* **Attestor laziness / collusion.** A delegate that never attests contributes nothing to any SN's score and is visible in
  `/api/masternodes/:id/attestations` and on the operator dashboard (`attestations sent / expected`); voters can act. A
  future SHIP MAY make attestation participation a forging requirement.
* **Chain-split safety.** `snRoot` is verified by every node from its own attestation table; a block with a wrong root is
  rejected like a bad `payloadHash`. Nodes missing attestations fetch them from ≥ 3 peers before judging; if they still
  cannot rebuild the root they accept the block under [SHIP-35.md](SHIP-35.md) finality certificates (≥ 15 delegates signed it) and log
  `SnRootUnverified`, exactly as light clients trust certificates ([SHIP-38.md](SHIP-38.md)).
* **Micro-payment fraud.** Off-chain by design; SNs should require pool inclusion for small sessions and block inclusion
  for large ones, and clients should prefer HTLC streaming for long sessions. Neither side can affect consensus.
* **Privacy.** Provider and messenger relays see traffic metadata; content is E2EE ([SHIP-40.md](SHIP-40.md)) and clients rotate relays
  per session. SNs MUST NOT log payload bodies; the manifest field `logging: none|metadata` is advertised and probed.

## Copyright

This document is placed in the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
