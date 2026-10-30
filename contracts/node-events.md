---
title: "Node events"
parent: Kontrakty
nav_order: 3
---

# Node events

**Status:** proposed
**Owners:** @IliaPoliak, @NotGeorgeLucas
**Drafted by:** @Ddystopia

Every node keeps an event log. The log holds observations of that one node
that never appear in blocks: losing proofs, rejections, timings, the mempool
and peers. The reader is pouw-indexer, which stores the events in Postgres
([ADR-0008]). Blocks are the source
of truth and events are observations. Anything derivable from blocks
(balances, inclusion, the winning proofs) the indexer takes from blocks.
Events carry only what blocks never contain.

Terms used below:

- **Event log**: an append-only list of events kept by one node.
- **Sequence number** (`seq`): a `u64` the node gives to each event. It
  starts at 1 and grows by 1.
- **Cursor**: the last `seq` the indexer has processed. The next request
  asks for events after it.
- **Long-poll**: a request that the node holds open until there is a new
  event or the wait time ends.
- **Losing proof**: a proof a miner made for the lottery of a block and
  that did not win. It is never written to a block.
- **Orphaned block**: a block that was accepted and later dropped because
  fork choice moved to another chain.
- **node_id**: a string that names the node which observed the event. It
  is set in the node configuration and does not change between restarts.

## Transport

The indexer pulls events from the node. The node does not push.

```text
GET /v1/events?after=<seq>&limit=<n>&wait_ms=<ms>
```

| parameter | meaning | default | maximum |
| --- | --- | --- | --- |
| `after` | return events with `seq` greater than this | 0 | none |
| `limit` | most events in one response | 100 | 1000 |
| `wait_ms` | how long to hold the request when nothing is new | 0 | 30000 |

A value above the maximum is lowered to the maximum. The response is:

```json
{ "events": [ { "seq": 41, "...": "..." } ], "next": 41 }
```

`next` is the `seq` of the last event in `events`. When `events` is empty
(the wait ended with nothing new), `next` equals `after`. The indexer sends
`next` as `after` in the following request.

### Ordering

- `seq` is strictly increasing per node.
- There are no gaps: the events of one node are exactly `1, 2, 3, ...`.
- There are no duplicates: one `seq` is one event, and a response never
  repeats an event the same request already returned.
- Events from different nodes have no common order. Each node has its own
  `seq`.

### Retention and resync

The node keeps at least the last `EVENT_RETENTION` events (100000) or the
events of the last 24 hours, whichever is more. A request with `after` older
than the oldest retained event gets HTTP 410:

```json
{ "error": { "code": "cursor_expired", "message": "oldest available seq is 52001",
             "details": { "oldest_seq": 52001 } } }
```

The indexer then:

1. Reads the missing blocks with `GET /v1/blocks?from=<h>&to=<h>`, from its
   last stored height to the head. Blocks cover everything derivable.
2. Sets its cursor to `error.details.oldest_seq - 1` and continues with
   `GET /v1/events`.

The observations in the dropped window are lost. This is acceptable
because events are observations.

### Indexer restart

The indexer stores the last processed `seq` for each `node_id`, in the same
Postgres transaction that stores the events. After a restart it reads that
value and sends it as `after`. A node restart does not reset `seq`: the node
persists the log, so the cursor stays valid. The first event after a start
is `node_started`.

### Why pull

- One direction of connection: the indexer opens it, so the node needs no
  address of the indexer and no retry logic.
- The indexer controls its pace and cannot be flooded.
- Restart is trivial: the cursor is the only state.
- [pouw-indexer#3] and
  [pouw-indexer#4] were
  closed with the note that the indexer uses endpoints defined in the node.

## Envelope

Every event has the same outer shape:

```json
{
  "seq": 41,
  "time": "2026-10-19T14:03:22.417Z",
  "node_id": "node-a",
  "kind": "tx_received",
  "data": {}
}
```

| field | type | meaning |
| --- | --- | --- |
| `seq` | u64 | sequence number of this node |
| `time` | string | RFC 3339 UTC with milliseconds, by the clock of the node |
| `node_id` | string | the node that observed the event |
| `kind` | string | lowercase snake_case name of the event kind |
| `data` | object | fields of the kind, see below |

Times observed by one node differ from another node's, so the indexer
stores `node_id` with every time (ADR-0008, timestamp question). The JSON
conventions are those of [node-api.md]: ids carry a type prefix
(`tx:`, `blk:`, `circ:`, `addr:`) and lowercase hex, byte blobs are base64,
field elements are `0x` and 62 hex digits, `u64` values are JSON numbers,
enum values are lowercase strings (`"groth16"`, `"plonk_kzg"`, `"marlin"`).
A proof hash is `sha256(borsh(ProofTx))` written as `proof:` and 64
lowercase hex digits. `reason_code` is a snake_case string, listed in the
[reason codes].

## Event kinds

### tx_received

The node accepted a transaction into its mempool, from a client or a peer.

| field | type |
| --- | --- |
| `txid` | `tx:` id |
| `tx_kind` | `"transfer"`, `"proof_request"` or `"register_circuit"` |
| `sender` | `addr:` id |
| `fee` | u64 |

```json
{ "seq": 41, "time": "2026-10-19T14:03:22.417Z", "node_id": "node-a",
  "kind": "tx_received",
  "data": { "txid": "tx:3f0c...e1", "tx_kind": "proof_request",
            "sender": "addr:9a41...07", "fee": 120 } }
```

### tx_rejected

The node refused a transaction. `txid` is absent when the bytes could not
be parsed.

| field | type |
| --- | --- |
| `txid` | `tx:` id, optional |
| `reason_code` | string |
| `message` | string |

```json
{ "seq": 42, "time": "2026-10-19T14:03:23.002Z", "node_id": "node-a",
  "kind": "tx_rejected",
  "data": { "txid": "tx:77ab...42", "reason_code": "bad_nonce",
            "message": "expected nonce 5, got 7" } }
```

### proof_request_started

This node is a miner and began proving a request.

| field | type |
| --- | --- |
| `txid` | `tx:` id of the proof request |
| `circuit` | `circ:` id |
| `system` | proof system |

```json
{ "seq": 43, "time": "2026-10-19T14:03:24.100Z", "node_id": "miner-1",
  "kind": "proof_request_started",
  "data": { "txid": "tx:3f0c...e1", "circuit": "circ:b8d2...90", "system": "groth16" } }
```

### proof_produced

This node finished a proof.

| field | type |
| --- | --- |
| `txid` | `tx:` id of the proof request |
| `circuit` | `circ:` id |
| `system` | proof system |
| `prove_ms` | u64, wall time of proving as measured by this node |
| `proof_hash` | `proof:` id |

```json
{ "seq": 44, "time": "2026-10-19T14:03:31.640Z", "node_id": "miner-1",
  "kind": "proof_produced",
  "data": { "txid": "tx:3f0c...e1", "circuit": "circ:b8d2...90", "system": "groth16",
            "prove_ms": 7512, "proof_hash": "proof:c41e...5d" } }
```

### lottery_result

The outcome of the lottery for one proof this node made. This event is the
source of wasted work: a row with `won: false` is a losing proof, and its
proof time comes from the `proof_produced` event with the same
`proof_hash`. Only the miner that made the attempt can report it, so the
wasted-work metric counts only cooperating miners.

| field | type |
| --- | --- |
| `txid` | `tx:` id of the proof request |
| `proof_hash` | `proof:` id |
| `circuit` | `circ:` id |
| `won` | bool |
| `eta` | field element |
| `mask` | field element |
| `usage_counter` | u64 |

```json
{ "seq": 45, "time": "2026-10-19T14:03:31.700Z", "node_id": "miner-1",
  "kind": "lottery_result",
  "data": { "txid": "tx:3f0c...e1", "proof_hash": "proof:c41e...5d", "circuit": "circ:b8d2...90",
            "won": false, "eta": "0x00a1...3c", "mask": "0x07f2...19", "usage_counter": 12 } }
```

### block_received

The node received a block from a peer, before verifying it.

| field | type |
| --- | --- |
| `hash` | `blk:` id |
| `height` | u64 |
| `from_peer` | string, peer id |

```json
{ "seq": 46, "time": "2026-10-19T14:03:33.010Z", "node_id": "node-a",
  "kind": "block_received",
  "data": { "hash": "blk:51de...a0", "height": 812, "from_peer": "peer-miner-1" } }
```

### block_verified

The node finished verifying a block, valid or not.

| field | type |
| --- | --- |
| `hash` | `blk:` id |
| `height` | u64 |
| `verify_ms` | u64, wall time of the whole verification |
| `proof_count` | u64, proof txs checked |
| `ok` | bool |

```json
{ "seq": 47, "time": "2026-10-19T14:03:33.190Z", "node_id": "node-a",
  "kind": "block_verified",
  "data": { "hash": "blk:51de...a0", "height": 812, "verify_ms": 168,
            "proof_count": 3, "ok": true } }
```

### block_accepted

The node added a block to its chain (its own or a peer's).

| field | type |
| --- | --- |
| `hash` | `blk:` id |
| `height` | u64 |
| `miner` | `addr:` id |
| `tx_count` | u64, entries of `txs` |
| `proof_count` | u64, entries of `proof_txs` |
| `bucket_count` | u32, `k` of ADR-0007 |
| `bucket_id` | u32 |

```json
{ "seq": 48, "time": "2026-10-19T14:03:33.205Z", "node_id": "node-a",
  "kind": "block_accepted",
  "data": { "hash": "blk:51de...a0", "height": 812, "miner": "addr:c0d3...18",
            "tx_count": 4, "proof_count": 3, "bucket_count": 8, "bucket_id": 5 } }
```

### block_rejected

The node refused a block. `height` is absent when the header could not be
read.

| field | type |
| --- | --- |
| `hash` | `blk:` id |
| `height` | u64, optional |
| `reason_code` | string |

```json
{ "seq": 49, "time": "2026-10-19T14:03:40.330Z", "node_id": "node-a",
  "kind": "block_rejected",
  "data": { "hash": "blk:0b9c...f4", "height": 813, "reason_code": "invalid_proof" } }
```

### block_orphaned

Fork choice moved away from an accepted block.

| field | type |
| --- | --- |
| `hash` | `blk:` id |
| `height` | u64 |

```json
{ "seq": 50, "time": "2026-10-19T14:03:52.480Z", "node_id": "node-a",
  "kind": "block_orphaned",
  "data": { "hash": "blk:51de...a0", "height": 812 } }
```

### proof_verified

One event for each proof tx during block verification.

| field | type |
| --- | --- |
| `txid` | `tx:` id of the proof request |
| `proof_hash` | `proof:` id |
| `system` | proof system |
| `verify_ms` | u64, wall time of this proof's verification |
| `ok` | bool |

```json
{ "seq": 51, "time": "2026-10-19T14:03:33.120Z", "node_id": "node-a",
  "kind": "proof_verified",
  "data": { "txid": "tx:3f0c...e1", "proof_hash": "proof:c41e...5d", "system": "groth16",
            "verify_ms": 4, "ok": true } }
```

### mempool_snapshot

The node emits it every 10 seconds, also when nothing changed.

| field | type |
| --- | --- |
| `pending_txs` | u64, all pending txs |
| `pending_proof_requests` | u64, the proof requests among them |
| `by_circuit` | object, `circ:` id to u64 pending proof requests |

```json
{ "seq": 52, "time": "2026-10-19T14:03:40.000Z", "node_id": "node-a",
  "kind": "mempool_snapshot",
  "data": { "pending_txs": 9, "pending_proof_requests": 7,
            "by_circuit": { "circ:b8d2...90": 5, "circ:1c77...ee": 2 } } }
```

### peer_connected and peer_disconnected

A peer connection opened or closed.

| field | type |
| --- | --- |
| `peer_id` | string |

```json
{ "seq": 53, "time": "2026-10-19T14:03:41.900Z", "node_id": "node-a",
  "kind": "peer_connected", "data": { "peer_id": "peer-miner-1" } }
```

### node_started

The node process started. It is the first event after every start.

| field | type |
| --- | --- |
| `chain_id` | u32 |
| `version` | string, node version |
| `height` | u64, chain height at start |

```json
{ "seq": 54, "time": "2026-10-19T14:00:00.015Z", "node_id": "node-a",
  "kind": "node_started",
  "data": { "chain_id": 1001, "version": "0.1.0", "height": 811 } }
```

### Reason codes

`reason_code` values are lowercase snake_case. The set starts with the
rejection reasons of [node-api.md]: `malformed`,
`bad_signature`, `bad_chain_id`, `bad_nonce`, `insufficient_balance`,
`unknown_circuit`, `over_limit` for transactions, and `invalid_proof`,
`bad_tx_root`, `bad_prev_hash`, `bad_bucket`, `over_limit` for blocks. A
consumer treats an unknown code as a plain string.

## JSON Schema

JSON Schema draft 2020-12 for one event. A response body is
`{events: [Event], next: u64}`.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://pouw-zk.github.io/pouw-docs/contracts/node-events.schema.json",
  "title": "Event",
  "type": "object",
  "required": ["seq", "time", "node_id", "kind", "data"],
  "properties": {
    "seq": { "type": "integer", "minimum": 1 },
    "time": { "type": "string", "format": "date-time" },
    "node_id": { "type": "string", "minLength": 1 }
  },
  "oneOf": [
    { "properties": { "kind": { "const": "tx_received" }, "data": { "$ref": "#/$defs/tx_received" } } },
    { "properties": { "kind": { "const": "tx_rejected" }, "data": { "$ref": "#/$defs/tx_rejected" } } },
    { "properties": { "kind": { "const": "proof_request_started" }, "data": { "$ref": "#/$defs/proof_request_started" } } },
    { "properties": { "kind": { "const": "proof_produced" }, "data": { "$ref": "#/$defs/proof_produced" } } },
    { "properties": { "kind": { "const": "lottery_result" }, "data": { "$ref": "#/$defs/lottery_result" } } },
    { "properties": { "kind": { "const": "block_received" }, "data": { "$ref": "#/$defs/block_received" } } },
    { "properties": { "kind": { "const": "block_verified" }, "data": { "$ref": "#/$defs/block_verified" } } },
    { "properties": { "kind": { "const": "block_accepted" }, "data": { "$ref": "#/$defs/block_accepted" } } },
    { "properties": { "kind": { "const": "block_rejected" }, "data": { "$ref": "#/$defs/block_rejected" } } },
    { "properties": { "kind": { "const": "block_orphaned" }, "data": { "$ref": "#/$defs/block_orphaned" } } },
    { "properties": { "kind": { "const": "proof_verified" }, "data": { "$ref": "#/$defs/proof_verified" } } },
    { "properties": { "kind": { "const": "mempool_snapshot" }, "data": { "$ref": "#/$defs/mempool_snapshot" } } },
    { "properties": { "kind": { "const": "peer_connected" }, "data": { "$ref": "#/$defs/peer" } } },
    { "properties": { "kind": { "const": "peer_disconnected" }, "data": { "$ref": "#/$defs/peer" } } },
    { "properties": { "kind": { "const": "node_started" }, "data": { "$ref": "#/$defs/node_started" } } }
  ],
  "$defs": {
    "TxId": { "type": "string", "pattern": "^tx:[0-9a-f]{64}$" },
    "BlockHash": { "type": "string", "pattern": "^blk:[0-9a-f]{64}$" },
    "CircuitId": { "type": "string", "pattern": "^circ:[0-9a-f]{64}$" },
    "Address": { "type": "string", "pattern": "^addr:[0-9a-f]{64}$" },
    "ProofHash": { "type": "string", "pattern": "^proof:[0-9a-f]{64}$" },
    "FieldElement": { "type": "string", "pattern": "^0x[0-9a-f]{62}$" },
    "U64": { "type": "integer", "minimum": 0 },
    "ProofSystem": { "enum": ["groth16", "plonk_kzg", "marlin"] },
    "ReasonCode": { "type": "string", "pattern": "^[a-z][a-z0-9_]*$" },
    "tx_received": {
      "type": "object", "additionalProperties": false,
      "required": ["txid", "tx_kind", "sender", "fee"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "tx_kind": { "enum": ["transfer", "proof_request", "register_circuit"] },
        "sender": { "$ref": "#/$defs/Address" },
        "fee": { "$ref": "#/$defs/U64" }
      }
    },
    "tx_rejected": {
      "type": "object", "additionalProperties": false,
      "required": ["reason_code", "message"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "reason_code": { "$ref": "#/$defs/ReasonCode" },
        "message": { "type": "string" }
      }
    },
    "proof_request_started": {
      "type": "object", "additionalProperties": false,
      "required": ["txid", "circuit", "system"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "circuit": { "$ref": "#/$defs/CircuitId" },
        "system": { "$ref": "#/$defs/ProofSystem" }
      }
    },
    "proof_produced": {
      "type": "object", "additionalProperties": false,
      "required": ["txid", "circuit", "system", "prove_ms", "proof_hash"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "circuit": { "$ref": "#/$defs/CircuitId" },
        "system": { "$ref": "#/$defs/ProofSystem" },
        "prove_ms": { "$ref": "#/$defs/U64" },
        "proof_hash": { "$ref": "#/$defs/ProofHash" }
      }
    },
    "lottery_result": {
      "type": "object", "additionalProperties": false,
      "required": ["txid", "proof_hash", "circuit", "won", "eta", "mask", "usage_counter"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "proof_hash": { "$ref": "#/$defs/ProofHash" },
        "circuit": { "$ref": "#/$defs/CircuitId" },
        "won": { "type": "boolean" },
        "eta": { "$ref": "#/$defs/FieldElement" },
        "mask": { "$ref": "#/$defs/FieldElement" },
        "usage_counter": { "$ref": "#/$defs/U64" }
      }
    },
    "block_received": {
      "type": "object", "additionalProperties": false,
      "required": ["hash", "height", "from_peer"],
      "properties": {
        "hash": { "$ref": "#/$defs/BlockHash" },
        "height": { "$ref": "#/$defs/U64" },
        "from_peer": { "type": "string" }
      }
    },
    "block_verified": {
      "type": "object", "additionalProperties": false,
      "required": ["hash", "height", "verify_ms", "proof_count", "ok"],
      "properties": {
        "hash": { "$ref": "#/$defs/BlockHash" },
        "height": { "$ref": "#/$defs/U64" },
        "verify_ms": { "$ref": "#/$defs/U64" },
        "proof_count": { "$ref": "#/$defs/U64" },
        "ok": { "type": "boolean" }
      }
    },
    "block_accepted": {
      "type": "object", "additionalProperties": false,
      "required": ["hash", "height", "miner", "tx_count", "proof_count", "bucket_count", "bucket_id"],
      "properties": {
        "hash": { "$ref": "#/$defs/BlockHash" },
        "height": { "$ref": "#/$defs/U64" },
        "miner": { "$ref": "#/$defs/Address" },
        "tx_count": { "$ref": "#/$defs/U64" },
        "proof_count": { "$ref": "#/$defs/U64" },
        "bucket_count": { "type": "integer", "minimum": 1, "maximum": 4294967295 },
        "bucket_id": { "type": "integer", "minimum": 0, "maximum": 4294967295 }
      }
    },
    "block_rejected": {
      "type": "object", "additionalProperties": false,
      "required": ["hash", "reason_code"],
      "properties": {
        "hash": { "$ref": "#/$defs/BlockHash" },
        "height": { "$ref": "#/$defs/U64" },
        "reason_code": { "$ref": "#/$defs/ReasonCode" }
      }
    },
    "block_orphaned": {
      "type": "object", "additionalProperties": false,
      "required": ["hash", "height"],
      "properties": {
        "hash": { "$ref": "#/$defs/BlockHash" },
        "height": { "$ref": "#/$defs/U64" }
      }
    },
    "proof_verified": {
      "type": "object", "additionalProperties": false,
      "required": ["txid", "proof_hash", "system", "verify_ms", "ok"],
      "properties": {
        "txid": { "$ref": "#/$defs/TxId" },
        "proof_hash": { "$ref": "#/$defs/ProofHash" },
        "system": { "$ref": "#/$defs/ProofSystem" },
        "verify_ms": { "$ref": "#/$defs/U64" },
        "ok": { "type": "boolean" }
      }
    },
    "mempool_snapshot": {
      "type": "object", "additionalProperties": false,
      "required": ["pending_txs", "pending_proof_requests", "by_circuit"],
      "properties": {
        "pending_txs": { "$ref": "#/$defs/U64" },
        "pending_proof_requests": { "$ref": "#/$defs/U64" },
        "by_circuit": {
          "type": "object",
          "propertyNames": { "$ref": "#/$defs/CircuitId" },
          "additionalProperties": { "$ref": "#/$defs/U64" }
        }
      }
    },
    "peer": {
      "type": "object", "additionalProperties": false,
      "required": ["peer_id"],
      "properties": { "peer_id": { "type": "string", "minLength": 1 } }
    },
    "node_started": {
      "type": "object", "additionalProperties": false,
      "required": ["chain_id", "version", "height"],
      "properties": {
        "chain_id": { "type": "integer", "minimum": 0, "maximum": 4294967295 },
        "version": { "type": "string" },
        "height": { "$ref": "#/$defs/U64" }
      }
    }
  }
}
```

## Coverage of ADR-0008

| ADR-0008 item | Source |
| --- | --- |
| Blocks, transfers, balances, address history, search | from blocks |
| Proof txs per system, proof sizes, fee per constraint | from blocks |
| Circuit registry | from blocks |
| Winning proofs per bucket | from blocks |
| Usage counter, bucket count and id, eta, mask, proof-system id | from blocks (node-api.md returns the derived fields); also `lottery_result`, `block_accepted` |
| Losing proofs | `lottery_result` with `won: false` |
| Buckets per round, losers and near misses per round | `lottery_result`, `block_accepted` |
| Rejected proofs and transactions | `tx_rejected`, `block_rejected`, `proof_verified` with `ok: false` |
| Unanswered client requests | `tx_received` and `proof_request_started` without a matching block (derived by the indexer) |
| Orphaned blocks | `block_orphaned` |
| Setup progress (pending, running, done) | not covered, see open points |
| Mempool contents and pending count | `mempool_snapshot`, `tx_received` |
| Proof received time | `tx_received`, `block_received` |
| Proving time | `proof_produced` |
| Verification duration | `proof_verified`, `block_verified` |
| CPU, RAM, disk, network, peers, sync, gossip lag, histograms | from metrics (`/metrics`); peer events cover connections |
| Finality states, proving-time distribution, miner share, amortization indicators, search, pagination | derived by the indexer |

## Open points

- One combined stream or one stream per node: does the indexer want a single
  endpoint that merges the events of several nodes, or does it read each
  node itself? This draft is per node. Decider: @NotGeorgeLucas.
- Snapshot rate: is `mempool_snapshot` every 10 seconds the right rate?
  Decider: @NotGeorgeLucas.
- Miner-claimed prove time as a field of the proof transaction would give an
  on-chain time next to the observed `prove_ms`. It changes the consensus
  format, so it is a consensus question. Decider: @IliaPoliak (ADR-0008).
- Setup progress events (pending, running, done) have no kind yet. They need
  a setup stage in the node. Decider: @IliaPoliak.

## Links

- [node-api.md], the REST API that serves `GET /v1/events`.
- [block.md], Block and ProofTx.
- [ADR-0007], stored and derived fields.
- [ADR-0008], indexer and explorer.
- [pouw-indexer#8], design of events from the node.
- [pouw-dashboard#13], explorer and dashboard requirements.

[ADR-0008]: ../decisions/0008-indexer-a-explorer.md
[pouw-indexer#3]: https://github.com/PoUW-ZK/pouw-indexer/issues/3
[pouw-indexer#4]: https://github.com/PoUW-ZK/pouw-indexer/issues/4
[node-api.md]: node-api.md
[reason codes]: #reason-codes
[block.md]: block.md
[ADR-0007]: ../decisions/0007-ulozene-a-odvodene-polia.md
[pouw-indexer#8]: https://github.com/PoUW-ZK/pouw-indexer/issues/8
[pouw-dashboard#13]: https://github.com/PoUW-ZK/pouw-dashboard/issues/13
