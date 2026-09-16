```
  SHIP: 30
  Title: Chain Compression with Evolutionary (Genetic) Encoding Search
  Authors: TechnoLog <technolog@smartholdem.io> / <SeZLuyhhYf2qxs4ArPJ71oEu3x8EsVw51C@sth>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core
  Created: 2026-05-27
  Last Update: 2026-06-19
```

## Abstract

Store historical blocks in **epochs** (e.g. 10 000 blocks) encoded with a columnar, dictionary-based scheme whose parameters
(field order, dictionary sizes, delta/varint choices, entropy coder settings) are selected per epoch by a **genetic
algorithm** run off the consensus path. The encoding is lossless and self-describing: every node decodes any epoch with
the parameters embedded in its header, and consensus data (hashes, signatures) is never altered.

## Motivation

Blockchain data is highly repetitive (public keys, recipients, fees, timestamps in arithmetic progression) but the optimal
encoding differs by era: early blocks are sparse, token eras carry manifests, PQ eras carry 2.4 KB signatures. Fixed
compression (gzip on serialized bytes) yields ~2x; structure-aware encoding tuned per epoch is expected to yield 6–10x,
cutting full-node disk and snapshot bandwidth accordingly.

## Specification

### Epoch container

`epoch:<n> -> header ‖ columns`, header: `version u8 ‖ params (genome, ≤ 256 B) ‖ columnCount u8 ‖ (columnId u8 ‖ codec u8 ‖ len u32)*`.
Columns: block header fields, transaction header fields, per-type payload fields, signatures (stored raw - incompressible),
each with a codec chosen from `{raw, varint, delta, dictionary(k), rle, zstd(level, dict)}`.

### Genome

Ordered gene list: column permutation, codec per column, dictionary size per dictionary column, zstd level, shared-dictionary
id (dictionaries trained on the previous epoch, stored once in `dict:<id>`). Fitness = encoded bytes (primary), decode time
(constraint: < 50 ms per 1 000 blocks on the reference machine).

### Search

Run by any node (or a maintainer script) with a population of 64 genomes, tournament selection, crossover on the gene list,
mutation rate 3 %, 200 generations, time-boxed. The best genome is **not consensus**: each node may re-encode its own
history with any genome; the result must round-trip to identical bytes (verified by re-hashing every block on encode).
Maintainers publish good genomes with snapshots ([SHIP-6.md](SHIP-6.md)) so most nodes never run the search.

### Guarantees

- Lossless: `decode(encode(epoch)) == epoch` verified by block ids and `payloadHash`.
- Tail epochs (last 2) stay in the hot `b:`/`t:` format for fork rollback; only finalised history is compacted.
- API and sync read through one accessor; `GetBlocks` ([SHIP-4.md](SHIP-4.md)) may ship whole encoded epochs to peers that advertise the
  codec, giving bandwidth savings for sync too.

## Rationale

Choosing parameters by evolution (instead of hand-tuning) adapts automatically as transaction mix changes (tokens, v3, NFTs)
and needs no protocol change; keeping it off-consensus avoids any risk to validation.

## Expected results (to be measured)

Legacy-era epochs: 7–9x vs raw; token/manifest epochs: 4–6x; PQ epochs: 2–3x (signatures dominate) - hence the value of
signature aggregation research ([SHIP-36.md](SHIP-36.md) follow-up).

## Reference Implementation

Not started; prototype plan: `src/compact/{epoch,genome,search}.rs`, `sth-core compact --epochs N`.
