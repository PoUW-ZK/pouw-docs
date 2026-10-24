---
title: "Web API"
parent: Kontrakty
nav_order: 4
---

# Web API

**Status:** proposed
**Owners:** @v-riazantsev and @ymarochok (see [Kontrakty])
**Author of the draft:** @v-riazantsev

The web API is the GraphQL API of the explorer in
[pouw-dashboard](https://github.com/PoUW-ZK/pouw-dashboard). Hasura serves it
over the PostgreSQL database of pouw-indexer ([ADR-0009]). It is read-only and
serves the 7 screens of ADR-0009 from indexed blocks. In v0 nothing in it
comes from a running node.

The operations the frontend uses are in [web-api.graphql]. This page describes
the types they read, the conventions and the rules.

## Terms used below

- **View**: a stored SQL query that looks like a table. The web API is a set
  of views in the database schema `api`. Hasura tracks only these views, never
  the indexer's tables, so a change of the indexer schema changes a view, not
  the API.
- **Canonical block**: a block on the chain the node follows. A block dropped
  by fork choice is **orphaned**. Every row says whether its block is
  canonical, and lists show canonical rows only.
- **Subscription**: a GraphQL query that stays open over a WebSocket. Hasura
  runs it again about once a second and sends the result when it changed.
- **Keyset pagination**: the next page is asked as "rows after the last one I
  saw", not as "skip the first N rows", so pages stay correct while new blocks
  arrive.

## Calling it

| What          | Where                                                                       |
| ------------- | --------------------------------------------------------------------------- |
| Queries       | `POST /v1/graphql` with `{"query": "...", "variables": {...}}`              |
| Subscriptions | WebSocket on `/v1/graphql`, protocol `graphql-transport-ws` or `graphql-ws` |
| Health        | `GET /healthz`                                                              |

No header and no login are needed. A request without a token runs as the role
`anonymous`, which can only read. Apollo Client, urql and graphql-request all
work. A typed client can be generated with GraphQL Code Generator from the
running Hasura and [web-api.graphql].

## Conventions

- **Ids** are prefixed lowercase hex strings, as in [node-api.md]: `blk:`,
  `tx:`, `proof:`, `circ:`, `addr:` followed by 64 hex digits. Search and
  filters take the same strings.
- **Other hashes** (`tx_root`, `witness_sha256`) are 64 hex digits without a
  prefix. **Field elements** (`eta`, `mask`) are 62 hex digits without a
  prefix, the 31 bytes little-endian, as in [node-api.md].
- **Numbers.** `bigint`, `Int` and `numeric` are JSON numbers. The test
  network stays below 2^53, so JavaScript reads them without loss. `share` is
  a fraction between 0 and 1.
- **Times** (`timestamptz`) are ISO 8601 strings with an offset, for example
  `2026-10-19T14:00:36+00:00`. Block times are whole seconds.
- **Null** means "not known" or "does not apply", for example `recipient` of a
  circuit registration or `indexed_height` before the first block. The
  tables below say which fields can be null.
- **Proof systems** are `groth16`, `plonk_kzg` and `marlin`. **Transaction
  kinds** are `transfer`, `register_circuit` and `proof_request`.

## Pagination

Lists are newest first. Blocks are ordered by `height`, everything else by
`sort_key`, a string that is unique and grows with the chain. The frontend
never builds or reads a `sort_key`, it only passes one back:

1. The first page omits `$before`. The operation's default then starts at the
   newest row.
2. The next page passes `$before` = the `height` or `sort_key` of the last
   item of the previous page.
3. A page with fewer items than `$limit` is the last one.

`$limit` is at most 100. Hasura cuts any larger answer to 100 rows.

## Types

Hasura names each type after its view. The generated GraphQL schema marks
every view field as nullable, because a view does not carry `NOT NULL`. The
tables below say which fields can really be null; all others never are.

### `blocks`

One row per indexed block, canonical or orphaned.

| Field                | Type        | Meaning                                                       |
| -------------------- | ----------- | ------------------------------------------------------------- |
| `hash`               | String      | `blk:` id                                                     |
| `height`             | bigint      | block height, 0 is genesis                                    |
| `prev_hash`          | String      | `blk:` id of the parent, all zeros in genesis                 |
| `time`               | timestamptz | block time from the header                                    |
| `miner`              | String      | `addr:` id, receives the fees                                 |
| `difficulty`         | bigint      | from the header                                               |
| `bucket_count`       | Int         | bucket count k ([ADR-0007])                                   |
| `bucket_id`          | Int         | bucket of this block                                          |
| `tx_root`            | String      | Merkle root of the block's entries                            |
| `size_bytes`         | Int         | size of the block in bytes                                    |
| `canonical`          | Boolean     | false when orphaned                                           |
| `transfer_count`     | bigint      | transfers in the block                                        |
| `registration_count` | bigint      | circuit registrations in the block                            |
| `proof_count`        | bigint      | proof transactions in the block                               |
| `fee_total`          | bigint      | sum of all fees in the block                                  |
| `transfers`          | [transfers] | relationship: the block's transfers                           |
| `registrations`      | [circuits]  | relationship: circuits registered in the block                |
| `proofs`             | [proofs]    | relationship: the block's proof transactions, the proof chain |

### `transfers`

One row per transfer in a block. A transfer that sits in two competing blocks
has two rows, one of them not canonical.

| Field        | Type        | Meaning                              |
| ------------ | ----------- | ------------------------------------ |
| `txid`       | String      | `tx:` id                             |
| `block_hash` | String      | `blk:` id of the block that holds it |
| `height`     | bigint      | height of that block                 |
| `time`       | timestamptz | time of that block                   |
| `position`   | Int         | index in the block's transactions    |
| `canonical`  | Boolean     | whether that block is canonical      |
| `sender`     | String      | `addr:` id                           |
| `recipient`  | String      | `addr:` id                           |
| `amount`     | bigint      | coins transferred                    |
| `fee`        | bigint      | fee paid to the miner                |
| `nonce`      | bigint      | transactions the sender sent before  |
| `sort_key`   | String      | pagination key                       |
| `block`      | blocks      | relationship: the block              |

### `circuits`

One row per registered circuit. The circuit id is the txid of the
`register_circuit` transaction, so `txid` and `circuit_id` hold the same hex.

| Field                 | Type        | Meaning                                                   |
| --------------------- | ----------- | --------------------------------------------------------- |
| `circuit_id`          | String      | `circ:` id                                                |
| `txid`                | String      | `tx:` id of the registration                              |
| `proof_system`        | String      | `groth16`, `plonk_kzg` or `marlin`                        |
| `constraint_count`    | bigint      | size of the circuit, the unit of useful work ([ADR-0006]) |
| `verifying_key_bytes` | Int         | size of the verifying key                                 |
| `circuit_bytes`       | Int         | size of the circuit                                       |
| `registrant`          | String      | `addr:` id                                                |
| `fee`                 | bigint      | fee of the registration                                   |
| `registered_block`    | String      | `blk:` id                                                 |
| `registered_height`   | bigint      | height of that block                                      |
| `registered_time`     | timestamptz | time of that block, null if the block is not indexed      |
| `canonical`           | Boolean     | whether the registering block is canonical                |
| `proof_count`         | bigint      | proofs for this circuit in canonical blocks               |
| `last_proof_height`   | bigint      | height of the newest of them, null before the first       |
| `useful_work`         | bigint      | `proof_count` times `constraint_count`                    |
| `sort_key`            | String      | pagination key                                            |
| `block`               | blocks      | relationship: the registering block                       |

### `proofs`

One row per proof transaction: a client's request and the proof that answered
it.

| Field                | Type        | Meaning                                                            |
| -------------------- | ----------- | ------------------------------------------------------------------ |
| `proof_hash`         | String      | `proof:` id                                                        |
| `request_txid`       | String      | `tx:` id of the proof request                                      |
| `block_hash`         | String      | `blk:` id of the block that holds it                               |
| `height`             | bigint      | height of that block                                               |
| `time`               | timestamptz | time of that block                                                 |
| `position`           | Int         | index in the block's proof chain, 0 is the first                   |
| `canonical`          | Boolean     | whether that block is canonical                                    |
| `miner`              | String      | `addr:` id of the miner who produced the proof                     |
| `client`             | String      | `addr:` id of the client who sent the request                      |
| `circuit_id`         | String      | `circ:` id                                                         |
| `proof_system`       | String      | from the circuit                                                   |
| `constraint_count`   | bigint      | from the circuit                                                   |
| `fee`                | bigint      | fee of the request                                                 |
| `nonce`              | bigint      | nonce of the request                                               |
| `usage_counter`      | bigint      | proofs by this miner for this circuit before this one ([ADR-0007]) |
| `proof_bytes`        | Int         | size of the proof                                                  |
| `witness_bytes`      | Int         | size of the witness inputs                                         |
| `witness_sha256`     | String      | hash of the witness inputs                                         |
| `public_input_count` | Int         | number of the client's public inputs                               |
| `eta`                | String      | integrity parameter, computed by the node ([block.md])             |
| `mask`               | String      | mask from miner, circuit and usage counter ([block.md])            |
| `sort_key`           | String      | pagination key                                                     |
| `block`              | blocks      | relationship: the block                                            |
| `circuit`            | circuits    | relationship: the circuit                                          |

The circuit receives `[eta, mask, ...client public inputs]`. The values of the
client's public inputs are not in the database in v0, only their number.

### `address_txs`

One row per transaction of an address, seen from that address, canonical
blocks only. Filter it by `address`.

| Field        | Type        | Meaning                                                        |
| ------------ | ----------- | -------------------------------------------------------------- |
| `address`    | String      | the `addr:` id this row belongs to                             |
| `direction`  | String      | `in`, `out`, or `self` for a transfer to itself                |
| `kind`       | String      | `transfer`, `register_circuit` or `proof_request`              |
| `txid`       | String      | `tx:` id                                                       |
| `height`     | bigint      | block height                                                   |
| `time`       | timestamptz | block time                                                     |
| `position`   | Int         | index in the block, null for a registration                    |
| `sender`     | String      | `addr:` id                                                     |
| `recipient`  | String      | `addr:` id, null except for a transfer                         |
| `amount`     | bigint      | null except for a transfer                                     |
| `fee`        | bigint      | fee paid                                                       |
| `nonce`      | bigint      | null for a registration                                        |
| `circuit_id` | String      | the new circuit, or the circuit asked for; null for a transfer |
| `proof_hash` | String      | the answering proof of a proof request, null otherwise         |
| `sort_key`   | String      | pagination key                                                 |

`position` and `nonce` of a registration are null because the database does
not store them yet.

### `addresses`

One row per address that a canonical block involves.

| Field                 | Type   | Meaning                                 |
| --------------------- | ------ | --------------------------------------- |
| `address`             | String | `addr:` id                              |
| `txs_sent`            | bigint | transfers and registrations sent        |
| `txs_received`        | bigint | transfers received from another address |
| `proof_requests`      | bigint | proof requests sent and answered        |
| `circuits_registered` | bigint | circuits registered                     |
| `blocks_mined`        | bigint | canonical blocks mined                  |
| `first_seen_height`   | bigint | first block that involves the address   |

There is no balance in v0, see [After v0](#after-v0).

### `stats`

Always exactly one row, totals over canonical blocks.

| Field                  | Type        | Meaning                                               |
| ---------------------- | ----------- | ----------------------------------------------------- |
| `indexed_height`       | bigint      | height of the newest canonical block, null when empty |
| `latest_block_time`    | timestamptz | its time, null when empty                             |
| `bucket_count`         | Int         | bucket count k of the newest block, null when empty   |
| `block_count`          | bigint      | canonical blocks                                      |
| `transfer_count`       | bigint      | transfers in them                                     |
| `proof_count`          | bigint      | proof transactions in them                            |
| `circuit_count`        | bigint      | circuits registered in them                           |
| `avg_block_interval_s` | float8      | mean seconds between blocks, null with fewer than two |

### `miner_shares` and `useful_work`

`miner_shares` has one row per miner of canonical blocks: `miner` (`addr:`
id), `blocks` (bigint), `share` (numeric, fraction of all canonical blocks, 4
decimals) and `last_block_height` (bigint).

`useful_work` has exactly one row per proof system: `proof_system`,
`proof_count` (bigint, proofs in canonical blocks) and `useful_work` (bigint,
sum of `constraint_count` over them). A system without proofs has zeros.

### `search`

`search(args: {q: "..."})` answers a list of hits with `kind` (`block`,
`transfer`, `proof`, `circuit`, `address`), `id` (the prefixed id to open) and
`height` (null for nothing). The prefix of `q` selects what is searched,
digits mean a block height. A `tx:` id finds a transfer, a registration (kind
`circuit`) or a proof request (kind `proof`). No match, or input that is no
id at all, answers an empty list.

## Operations by screen

All operations are in [web-api.graphql].

| Screen               | Operations                               | Live                  |
| -------------------- | ---------------------------------------- | --------------------- |
| 1. Home              | `HomeStats`, `LatestBlocks`, `Search`    | the two subscriptions |
| 2. Block             | `BlockByHeight`, `BlockByHash`, `Blocks` | no                    |
| 3. Transaction       | `Transaction`                            | no                    |
| 4. Proof transaction | `Proof`                                  | no                    |
| 5. Circuits          | `Circuits`, `Circuit`, `CircuitProofs`   | no                    |
| 6. Address           | `Address`, `AddressTxs`, `MinedBlocks`   | no                    |
| 7. Mining            | `Mining`                                 | no                    |

Rules for the operations:

- `/block/<height>` uses `BlockByHeight`, which answers the canonical block
  only. `/block/blk:...` uses `BlockByHash`, which answers the block also when
  it is orphaned, so old links keep working.
- `Transaction` asks all three kinds at once, because a txid alone does not
  say its kind. Exactly one of the three lists is not empty for a known txid,
  and all three are empty for an unknown one. A proof request links to screen
  4 through `proof_hash`.
- An empty list from `Address`, `BlockByHeight` and the other detail queries
  means "not found". GraphQL has no 404.
- Any query can be a subscription, with the same selection. The two
  subscriptions listed above are the ones the home screen needs; other screens
  load once.

## Limits and permissions

- The role `anonymous` can select every field of every view above, and
  nothing else: no mutations, no aggregates, no access to the indexer tables.
- Every answer has at most 100 rows per list.
- Writes are impossible: a mutation answers an error, because none exists for
  the role.

## Local development and mock

The frontend runs against a local Hasura with fake data, which is the mock of
this API. pouw-dashboard starts PostgreSQL and Hasura with the views and the
seed data from `hasura/seeds/`. The seed is one small consistent chain:
blocks 0 to 3, an orphaned competitor at height 3, two circuits, transfers and
proofs that refer to each other. The Hasura console at
<http://localhost:8081/console> shows the schema and runs any operation. How
to start it is in the README of pouw-dashboard.

## Changes

The views are the contract. Adding a view, a field or a relationship keeps
existing operations working, so it needs no announcement beyond this page.
Removing or renaming a field, or changing its meaning, breaks the frontend:
it needs a pull request that changes this page, [web-api.graphql] and the
frontend together, reviewed by both owners. While the status is `proposed`,
the contract changes in place.

## After v0

When nodes run, the indexer stores more in PostgreSQL ([ADR-0008],
[node-events.md]), and the API grows by new views and fields:

- mempool counters and their history, and pending transactions, from
  `mempool_snapshot` and `tx_received` events;
- balance and nonce of an address, copied by the indexer from the node;
- verification and proving times, losing proofs and the lottery breakdown of
  a block.

These are additions, so the operations above keep working.

## Links

- [web-api.graphql], the operations of this API.
- [ADR-0009], the explorer screens and the data architecture.
- [ADR-0008], indexer and explorer.
- [block.md], the block format, eta and mask.
- [node-api.md] and [node-events.md], what a node will offer.

[Kontrakty]: index.md
[web-api.graphql]: web-api.graphql
[node-api.md]: node-api.md
[node-events.md]: node-events.md
[block.md]: block.md
[ADR-0006]: ../decisions/0006-tri-dokazovacie-systemy.md
[ADR-0007]: ../decisions/0007-ulozene-a-odvodene-polia.md
[ADR-0008]: ../decisions/0008-indexer-a-explorer.md
[ADR-0009]: ../decisions/0009-explorer-v0-scope.md
