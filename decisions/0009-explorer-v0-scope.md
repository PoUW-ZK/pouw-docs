---
title: "ADR-0009: PoUW Explorer v0 Scope and Screen Catalog"
parent: Rozhodnutia (ADR)
nav_order: 9
---

# ADR-0009: PoUW Explorer v0 Scope, Screen Specifications, and Data Architecture

**Status:** proposed  
**Date:** 2026-10-17  
**Author:** @v-riazantsev

## Context

Following [**ADR-0008**](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0008-indexer-a-explorer.md), [pouw-dashboard](https://github.com/PoUW-ZK/pouw-dashboard) adopts a read-only architecture powered by PostgreSQL and Hasura (GraphQL). We must define the concrete functional scope, screen catalog, and data sourcing rules for the **v0 prototype** of the block explorer.

We need to establish clear boundaries for what can be built immediately using finalized block data and indexer states, versus what telemetry relies on unreleased node event streams or transient mempool staging.

## Options

1. **Complete Scope in v0 (Finalized Blocks + Live Mempool + Node Events)**
   - **Pros:** Delivers all features from [pouw-dashboard#13](https://github.com/PoUW-ZK/pouw-dashboard/issues/13) immediately, including real-time mempool queues, losing proof breakdowns, and verification latency metrics.
   - **Cons:** Blocks v0 delivery until node event stream is implemented. Storing high-churn transient mempool data in PostgreSQL creates database bloat and write amplification.

2. **Decoupled Block-Focused Scope for v0 (Recommended)**
   - **Pros:** Frontend development ([pouw-dashboard](https://github.com/PoUW-ZK/pouw-dashboard)) can start immediately using PostgreSQL and Hasura. Uses finalized block data for block history, account balances, circuit registries, and miner power share. Keeps the system lightweight and secure.
   - **Cons:** Defers live mempool tracking (`/mempool`), losing proof analytics, and node-side timing distributions to post-v0 milestones.

3. **Direct Node RPC Polling from the Frontend**
   - **Pros:** Obtains live node metrics (such as active worker counts or pending pool depth) without waiting for indexer DB schema updates.
   - **Cons:** Violates the [ADR-0008](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0008-indexer-a-explorer.md) architecture. Exposes private node RPC endpoints to public browsers and creates inconsistent data fetching layers on the client.

## Decision

We adopt **Option 2**. The [pouw-dashboard](https://github.com/PoUW-ZK/pouw-dashboard) v0 release will consist of a 7-screen scope built strictly over PostgreSQL and Hasura GraphQL subscriptions. Live node RPC metrics will be polled by [pouw-indexer](https://github.com/PoUW-ZK/pouw-indexer) and served through PostgreSQL status tables rather than direct frontend RPC calls. Mempool tracking and non-block event research metrics are formally deferred.

---

### Explorer v0 Screen Specifications

#### 1. Home / Overview Dashboard

- **Purpose:** Primary landing view providing a live macro snapshot of the network and protocol statistics.
- **Content & Fields:**
  - **Latest Blocks Feed:** Real-time list showing `height`, `hash`, `timestamp`, `miner` address, and transaction count. Sourced from finalized blocks in PostgreSQL.
  - **Network Status Card:** Current `indexed_height`, active bucket count `k` (parsed directly from the latest block header), total registered circuits count, and total processed proof transactions count.
  - **Global Search Bar:** Universal lookup querying PostgreSQL by block height/hash, transaction hash, address, or circuit ID.
- **Data Source:** Indexed continuously by [pouw-indexer](https://github.com/PoUW-ZK/pouw-indexer) from block headers into PostgreSQL and served via Hasura WebSocket subscriptions.
- **Deferred (Post-v0):** Live mempool queue counters, and global aggregate tickers for Useful vs. Wasted Work.

#### 2. Block Detail Screen (`/block/:hash_or_height`)

- **Purpose:** Detailed inspection of a specific mined block and its packed transactions.
- **Content & Fields:**
  - **Header Metadata:** `height`, `hash`, `prev_block_hash`, `timestamp`, `miner`, and `difficulty`.
  - **PoUW Consensus Fields ([ADR-0007](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0007-ulozene-a-odvodene-polia.md)):** Stored bucket count $k$ and `bucket_id` (extracted from the block header).
  - **Transactions Lists (Tabbed / Separated):**
    - *Coin Transfers:* `tx_hash`, `from`, `to`, `amount`, `fee`.
    - *Block Proof Transactions (Proof Chain):* Itemized list of all proofs forming the winning miner's proof chain ($\pi_1 \dots \pi_n$), showing `tx_hash`, `circuit_id`, `fee`, `client`, and `usage_counter`.
- **Data Source:** Direct query against `blocks`, `coin_transactions`, and `proof_transactions` PostgreSQL tables via Hasura.
- **Deferred (Post-v0):** Collapsible "Lottery Breakdown" drawer displaying discarded losing proofs from competing miners, near-misses, and candidate trials for that round.

#### 3. Coin Transaction Detail Screen (`/tx/coin/:hash`)

- **Purpose:** Inspect standard value transfers and token movements.
- **Content & Fields:** `tx_hash`, `block_height`, `timestamp`, `from`, `to`, `amount`, and `fee`.
- **Data Source:** Direct query from the `coin_transactions` table in PostgreSQL.

#### 4. Proof Transaction Detail Screen (`/tx/proof/:hash`)

- **Purpose:** Inspect a zero-knowledge proof transaction performing useful work.
- **Content & Fields:**
  - **Identifiers:** `tx_hash`, `block_height`, `timestamp`, `client`.
  - **Proof Metadata:** `circuit_id` (clickable link navigating to Circuit Detail), `fee`, proof payload size.
  - **Consensus & State Fields ([ADR-0007](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0007-ulozene-a-odvodene-polia.md)):**
    - `usage_counter`: Stored integer in proof entry (miner/circuit nonce).
    - `integrity_parameter_eta`: Derived hash (`Hash(BlockHeader + Proofs)`) binding the proof to the block.
    - `derived_mask`: Derived value (`Hash(MinerAddress + UsageCounter)`) preventing work amortization.
  - **Verification:** `verification_status` (Valid/Invalid) and `public_inputs`.
- **Data Source:** `usage_counter` extracted from block data. Derived fields ($\eta$, Mask) computed by [pouw-indexer](https://github.com/PoUW-ZK/pouw-indexer) upon ingestion per [ADR-0007](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0007-ulozene-a-odvodene-polia.md). `proof_system_id` (Groth16, PLONK, Marlin) resolved via foreign key relationship to `circuits`.
- **Deferred (Post-v0):** Node-side verification duration (`verify_duration`), node receipt timestamp (`proof_received_at`), and client-vs-node proving time asymmetry ratios.

#### 5. Circuit Registry & Detail Screens

- **A. Circuit Registry Catalog (`/circuits`):** Catalog displaying `circuit_id`, `proof_system_id`, `constraint_count`, `owner`, and `registration_block_height`.
- **B. Circuit Detail Screen (`/circuit/:circuit_id`):** Deep-dive into a specific circuit showing metadata, total proofs verified count, and a list of all historical proof transactions submitted for it.
- **Data Source:** Populated from `circuits` and `proof_transactions` tables in PostgreSQL.

#### 6. Address / Account Detail Screen (`/address/:address`)

- **Purpose:** Inspect an individual account or miner history.
- **Content & Fields:** `balance`, total transaction count, incoming/outgoing transfer history, and miner block rewards history.
- **Data Source:** Executed as indexed SQL aggregation queries against PostgreSQL.

#### 7. Mining & Network Overview Screen (`/mining`)

- **Purpose:** Macro summary of miner participation and useful work metrics based on finalized blocks.
- **Content & Fields:**
  - **Miner Power Share (v0):** Percentage of finalized blocks mined per address (`COUNT(blocks) / total_blocks`).
  - **Block Intervals:** Average time between consecutive block timestamps.
  - **Useful Work (v0):** Aggregate constraint complexity of all proofs included in finalized blocks (`SUM(constraint_count)` across all block proof chain entries).
- **Data Source:** Aggregated via SQL queries over `blocks` and `proof_transactions` tables.
- **Deferred (Post-v0):** Proving-time distribution histograms (median, 95th percentile) and true Wasted Work metrics incorporating discarded losing proof trials from non-winning miners.

## Consequences

- **What becomes easier:** The frontend team (@ymarochok) can begin building all 7 screens immediately using standard GraphQL schema definitions over PostgreSQL without waiting for protocol node event streams.
- **What becomes harder:** Users will not have real-time visibility into unconfirmed mempool transactions or losing lottery attempts until post-v0 event streaming and staging layers are developed.

## Open Questions for Team Members

### For Indexer & Database (@NotGeorgeLucas)

1. **Derived Fields Ingestion ([ADR-0007](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0007-ulozene-a-odvodene-polia.md)):** Are we storing these derived fields (`integrity_parameter_eta` and `derived_mask`) as dedicated database columns during ingestion (whether computed by the node API or parsed/calculated by the indexer), so Hasura can serve them instantly?

### For Frontend (@ymarochok)

1. **API Layer Selection:** Is pure GraphQL over Hasura acceptable for [pouw-dashboard](https://github.com/PoUW-ZK/pouw-dashboard), or do you require Hasura's built-in REST endpoints (where Hasura automatically exposes REST routes mapped to stored GraphQL queries)?

### For Node & Protocol (@IliaPoliak / @Ddystopia)

1. **Consensus Fields Confirmation:** Confirming final agreement on [ADR-0007](https://github.com/PoUW-ZK/pouw-docs/blob/main/decisions/0007-ulozene-a-odvodene-polia.md) field requirements (bucket count $k$ and `bucket_id` in block headers, and `usage_counter` in each `ProofTx` entry).
