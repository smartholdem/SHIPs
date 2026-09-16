```
  SHIP: 35
  Title: BFT Finality Gadget over Iroh Gossip
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-06-08
  Last Update: 2026-09-01
```

## Abstract

Add explicit **finality** to the DPoS chain: the 21 active delegates sign short votes for block hashes over a dedicated gossip
topic; a block with signatures from **≥ 15 of 21** (> 2/3) delegates is *final* and can never be rolled back by any node.
Finality is reached within 1–2 slots instead of the ~3-minute probabilistic window, exchanges can credit deposits after one
block, and pruning ([SHIP-31.md](SHIP-31.md)) and light clients ([SHIP-38.md](SHIP-38.md)) get a hard anchor.

## Motivation

Today a block is "probably final" once 21 delegates have built on it - a full round (168 s). Rust nodes already roll back
forks automatically ([SHIP-2.md](SHIP-2.md)), which protects consistency but not user experience or exchange deposit times. The network has
authenticated, sub-100 ms fan-out ([SHIP-4.md](SHIP-4.md)) and a signed delegate identity ([SHIP-8.md](SHIP-8.md)): the ingredients of a BFT vote.

## Specification

### Votes

`FinalityVote { height, blockId, round, delegatePk, sig }` - `sig` = Schnorr (and, after [SHIP-22.md](SHIP-22.md), ML-DSA hybrid) over
`sha256("sth-finality-v1" ‖ height ‖ blockId)`. Gossip topic `finality` (derived from nethash). A delegate votes for a block
when it has validated it and it extends the delegate's last finalised block (no vote for two different blocks at one
height - provable equivocation).

### Finality

A **finality certificate** = ≥ 15 distinct active-delegate votes for `(height, blockId)`. Nodes store certificates
(`fc:<height>`), refuse to roll back below the highest certified height, and include the latest certificate in the next
block header extension (`finalityCert`, block v1), so it becomes part of the chain for late joiners and light clients.

### Liveness

Votes are optional for block production: if fewer than 15 delegates vote (partition, old software), the chain continues
under today's probabilistic rules and finality simply lags; when votes resume, the newest common block is certified.
Timeouts: a delegate re-broadcasts its vote for 2 slots; certificates for skipped heights are not required (finalising `h`
finalises all ancestors).

### Slashing (soft)

Equivocation (two votes at one height) is recorded on chain by anyone (`EquivocationProof` transaction, fee refunded);
the delegate is marked and excluded from the active set for 30 rounds - voters see it on the Delegate Dashboard.

### Parameters

`finality: { active, quorum: 15, voteTopic, certInHeader }` milestone; `quorum` fixed relative to `activeDelegates = 21`.

## Rationale

Voting on blocks that already exist (a "gadget", as in Casper FFG / GRANDPA) needs no change to slot scheduling or forging
and degrades gracefully; > 2/3 of 21 tolerates up to 6 faulty or offline delegates.

## Backwards Compatibility

Votes are gossip-only until `certInHeader` (block v1); legacy nodes are unaffected but cannot benefit.

## Reference Implementation

Not started; design notes in `docs/CONSENSUS.md §2a`.
