```markdown
  SHIP: TBD
  Title: Pruned Child DEX Database and Token-to-Token Market Abstraction
  Authors: TechnoLog <technolog@smartholdem.io>
  Status: Draft
  Discussions-To: https://github.com/smartholdem/SHIPs/issues
  Type: Standards Track
  Category: Core / Market
  Created: 2026-11-24
  Last Update: 2026-11-24
  Requires: 14, 16, 43

```

## Abstract

This SHIP specifies a dedicated, prunable child `sled` database (`dex.sled`) attached to `sth-core` for order book state, trade history, fills, and OHLCV candles, together with an asset-pair abstraction `(baseAsset, quoteAsset)` that supports both native `STH <> Token` markets and cross-token `Token <> Token` trading (e.g., `wUSDT <> Token`) without bloating the primary L1 blockchain state.

## Motivation

SmartHoldem's primary blockchain (`chain.sled`) maintains a compact footprint (~1.7GB across 12.5M blocks). Under high-frequency DEX activity, storing full order histories, fills, closed orders, and candlestick aggregations inside the consensus state introduces unnecessary I/O contention and long-term storage growth.

Separating market indexing and historical order records into an independent, deterministically reconstructable `dex.sled` database isolates disk I/O, enables configurable pruning of historical market data, and generalizes market pairs beyond `STH`-quoted markets to arbitrary `TokenA <> TokenB` pairs.

## Specification

### 1. Asset and Market Pair Model

#### 1.1 Asset Identifier (`AssetId`)

An asset in the DEX layer is represented as a tagged union:

* `NativeSth` - the native `STH` coin (API representation: `"STH"`).
* `Token([u8; 32])` - a native `typeGroup: 3` token identified by its 32-byte registration ID (64-character hex string).

#### 1.2 Market Pair (`MarketPair`)

A trading market is defined by an ordered pair `(base: AssetId, quote: AssetId)` where `base != quote`:

* **Existing `STH <> Token` markets ([SHIP-14.md](SHIP-14.md) / [SHIP-43.md](SHIP-43.md)):** mapped to `base = Token(tokenId)`, `quote = NativeSth`.
* **Cross-token `Token <> Token` markets:** mapped to `base = Token(baseTokenId)`, `quote = Token(quoteTokenId)` with canonical pair ordering and decimal-aware price scaling (`baseDecimals`, `quoteDecimals`).

### 2. Storage Separation (`chain.sled` vs. `dex.sled`)

| Storage Layer | Responsibilities | Lifecycle & Pruning Policy |
| --- | --- | --- |
| **Primary Chain (`chain.sled`)** | Consensus balances (`STH` and tokens), token registries, active escrow locks required for transaction validation | Closed and cancelled orders are removed from active consensus state once past the finalization/undo window |
| **Child DEX DB (`dex.sled`)** | Open order book index, full order metadata (`fills`, status), trade history (`trades`), OHLCV candles, block sync checkpoints | Configurable background pruning by block age/height and per-pair retention floor; fully reconstructable from chain blocks |

### 3. `dex.sled` Key Space

All multi-byte integers in keys MUST use big-endian (`BE`) encoding to preserve lexicographical ordering during range scans:

* `meta:last_height` -> `u64 LE` (last indexed block height).
* `meta:last_block_id` -> `[u8; 32]` (block hash at `last_height` for reorg detection).
* `orders:meta:<order_id>` -> binary-encoded order record (`orderId`, `pair`, `side`, `maker`, `price`, `priceUnit`, `amount`, `filled`, `remaining`, `open`, `createdHeight`, `closedHeight`, `timestamp`, `fills[]`).
* `orders:open:<pair_key>:<side>:<price_be>:<order_id>` -> empty value / compact pointer (enables price-sorted order book iteration).
* `orders:maker:<maker_addr>:<order_id>` -> index of orders created by `maker_addr`.
* `trades:<pair_key>:<height_be>:<tx_idx_be>:<trade_id>` -> trade record (`txId`, `orderId`, `pair`, `maker`, `taker`, `baseAmount`, `quoteAmount`, `price`, `height`, `timestamp`).
* `candles:<pair_key>:<tf_code>:<bucket_be>` -> aggregated OHLCV candle (`open`, `high`, `low`, `close`, `baseVolume`, `quoteVolume`, `tradesCount`).

### 4. Indexing, Reorg Handling, and Reconstruction

1. **Block Application:** On each committed block at height `H`, the DEX indexer processes market transactions (`TokenSell`, `TokenSellV2`, `TokenBuy`, `TokenCancel`, and future `Token <> Token` order types) and updates `dex.sled` atomically via `sled::Batch`, advancing `meta:last_height` and `meta:last_block_id`.
2. **Rollback / Reorg:** `dex.sled` maintains an undo journal or periodic state snapshots bounded by the finality window. On chain rollback to height `R`, `dex.sled` reverts state back to `R` (or rebuilds from the nearest valid checkpoint).
3. **Full Reconstruction:** Nodes MUST provide a `rebuild_dex(from_height)` routine capable of deterministically reconstructing `dex.sled` from canonical block history if the child database is missing, corrupted, or upgraded to a new schema version.

### 5. Pruning Policy

```yaml
dex:
  enabled: true
  path: "dex.sled"
  pruning:
    enabled: true
    keep_closed_orders_blocks: 324000   # ~30 days at 8s block time
    keep_trades_blocks: 324000
    min_keep_trades_per_pair: 1000
    prune_interval_blocks: 100

```

* **Active Orders:** Open orders (`open == true`) MUST NEVER be pruned.
* **Closed / Cancelled Orders:** Orders with `open == false` and `closedHeight < currentHeight - keep_closed_orders_blocks` are deleted in batches (`sled::Batch`).
* **Trades & Candles:** Trades older than `keep_trades_blocks` are pruned via prefix range scans, except that the most recent `min_keep_trades_per_pair` trades per market pair MUST be retained so low-volume markets preserve chart and feed continuity. Coarse-grained candles (`1d`, `1w`) MAY be retained longer than fine-grained candles (`1m`, `5m`).

### 6. API Compatibility

Existing REST endpoints continue to operate transparently by reading from `dex.sled`:

* `GET /api/tokens/:key/orders`
* `GET /api/tokens/:key/orders/:orderId`
* `GET /api/tokens/:key/trades`
* `GET /api/wallets/:address/orders`

Pair-generalized endpoints (`GET /api/dex/markets/:base/:quote/orders`, `/trades`, `/candles`) expose the same underlying `MarketPair` storage layout for `Token <> Token` pairs.

## Rationale

* **Dedicated `sled` instance:** Separating `dex.sled` from `chain.sled` prevents write amplification and compaction/GC overhead from frequent order churn from affecting block validation latency.
* **Lexicographical `BE` key design:** Encoding prices, heights, and time buckets in big-endian allows `sled` prefix iterators to serve sorted order books, paginated trade feeds, and range-based pruning without secondary in-memory sorting.
* **Retention floor (`min_keep_trades_per_pair`):** Pure height-based pruning would wipe history for infrequently traded pairs; combining height age with a minimum trade count keeps market UIs populated.
* **Upfront `(base, quote)` abstraction:** Modeling `STH` as `AssetId::NativeSth` alongside `AssetId::Token` avoids a breaking storage migration when cross-token pairs (`wUSDT <> Token`) are introduced.

## Backwards Compatibility

This specification does not alter consensus rules, transaction formats, or wire serialization. Nodes with `dex.enabled: false` continue validating blocks normally while disabling local DEX indexing and market query endpoints. Existing `STH <> Token` API responses remain unchanged.

## Reference Implementation

Not started. Planned modules in `sth-core`: `src/dex/{mod.rs, types.rs, store.rs, indexer.rs, pruner.rs, queries.rs}`.

## Security Considerations

* **State Integrity:** `dex.sled` is a non-consensus derived index; consensus balance transfers and escrow checks remain strictly enforced in `chain.sled`.
* **Reorg Consistency:** Verifying `meta:last_block_id` on startup and binding undo records to block hashes prevents stale or orphaned order states across chain reorganizations.
* **Disk & I/O DoS Mitigation:** Batched pruning (`sled::Batch`) prevents long write stalls, while protocol-level order limits (`maxOpenOrders`) bound the unprunable active order set.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
