---
title: "Node API"
parent: Kontrakty
nav_order: 2
---

# Node API

**Status:** proposed
**Owners:** @IliaPoliak and @Quolen (see [Kontrakty])
**Author of the draft:** @Ddystopia

A PoUW-ZK full node serves this API. Clients submit transactions through it,
the indexer reads the chain through it, light clients read headers and
inclusion proofs, and operators read health and metrics. It is REST. Transactions go in as borsh
bytes or as JSON, and blocks come out as borsh bytes or as a JSON view for
people and tools.
This is not a production system. The goal is to reduce friction between six
repositories in four languages, so the simpler technology wins. gRPC would need
code generation in every language, and protobuf is not a canonical encoding, so
it could not replace borsh for hashing anyway.

The machine-readable form is [node-api.yaml] (OpenAPI 3.1). The
block format is in [block.md] and the event log in
[node-events.md].

## Terms used below

- A **txid** is the identifier of a transaction: SHA-256 of the borsh bytes of
  the signed transaction.
- The **mempool** is the set of valid transactions a node has accepted and that
  no block includes yet.
- An **inclusion proof** is the list of sibling hashes that lets anyone
  recompute the Merkle root of a block from one leaf and compare it with the
  root in the header.
- **Long-poll** means the server holds a request open for some time when it has
  nothing to answer, and answers as soon as there is something new.
- A **derived field** is a value that a block does not store and that the node
  computes from the block and the registry (ADR-0007). The API returns it
  computed.
- A **light client** keeps only block headers and asks a full node for
  everything else, then checks the answer against the headers.

## Conventions

These rules apply to the JSON view and to the error bodies.

- **Ids** are strings with a type prefix and 64 lowercase hex digits:
  `tx:<64 hex>`, `blk:<64 hex>`, `circ:<64 hex>`, `addr:<64 hex>`,
  `proof:<64 hex>` (the hash of a proof tx). A block hash
  pasted where a txid belongs fails at parse. The OpenAPI file has one named
  schema per type (`TxId`, `BlockHash`, `CircuitId`, `Address`, `ProofHash`)
  with the pattern `^tx:[0-9a-f]{64}$`, `^blk:[0-9a-f]{64}$`,
  `^circ:[0-9a-f]{64}$`, `^addr:[0-9a-f]{64}$` and `^proof:[0-9a-f]{64}$`.
- **Other hashes** (`tx_root` and the inclusion proof siblings) are 64
  lowercase hex digits without a prefix.
- **Byte blobs** (`witness`, `proof`, `verifying_key`, `circuit`, `signature`)
  are base64 strings.
- **Field elements** (`public_inputs`, `eta`, `mask`) are 62 lowercase hex
  digits, without a `0x` prefix: the 31 bytes of the FieldInput verbatim, in
  the byte order of borsh (little-endian, like every borsh integer). No byte
  reordering between JSON and borsh.
- **Numbers.** u64 quantities are JSON numbers. The test network never
  exceeds 2^53.
- **Enum tags** are lowercase strings: `"transfer"`, `"proof_request"`,
  `"register_circuit"`, `"groth16"`, `"plonk_kzg"`, `"marlin"`.
- **Errors** have one shape: `{"error": {"code": "<snake_case>", "message": "..."}}`.
  HTTP 400 means malformed, 422 means rejected by the rules (bad signature, bad
  nonce, insufficient balance, unknown circuit, over a limit), 404 means an
  unknown id, 410 means an expired event cursor.
- **Borsh is the default.** Every GET of a block, header or tx answers
  `application/octet-stream` with the borsh bytes. `POST /v1/txs` takes
  `Content-Type: application/octet-stream` with the borsh bytes of a SignedTx,
  or `Content-Type: application/json` with the SignedTx view without `txid`.
  See [JSON view] for the JSON representation.
- **The OpenAPI file is the normative schema.** Every object is closed (no
  extra fields), every string has a pattern and every number a range. In Rust
  each id is a newtype whose parser rejects a string without its own prefix
  (see the types in [block.md]). Other languages are expected to do
  the same.

## Endpoints

All paths have the prefix `/v1` except `/health` and `/metrics`.

| Method | Path | Purpose | Called by |
| --- | --- | --- | --- |
| POST | `/v1/txs` | submit a signed transaction as borsh or JSON, answer 202 with the txid | clients, workload generator, light client |
| GET | `/v1/txs/{txid}` | the tx, its status and, for an included proof request, the derived fields | clients, indexer |
| GET | `/v1/txs/{txid}/inclusion-proof` | Merkle inclusion proof of an included tx | light clients |
| GET | `/v1/blocks/head` | header of the tip with `hash` | clients, light clients |
| GET | `/v1/blocks/{height_or_hash}` | full block, borsh or JSON view | indexer, explorer |
| GET | `/v1/blocks/{height_or_hash}/header` | header of one block | light clients |
| GET | `/v1/blocks?from=&to=` | blocks as a JSON list or length-prefixed borsh, at most 100 per call | indexer |
| GET | `/v1/headers?from=&to=` | headers as a JSON list, at most 1000 per call | light clients |
| GET | `/v1/circuits` | registry entries, without blobs unless `?include=keys` | clients, workload generator |
| GET | `/v1/circuits/{circuit_id}` | one registry entry | clients, miners |
| GET | `/v1/accounts/{addr}` | balance and nonce | clients, light clients |
| GET | `/v1/mempool` | pending txids and counts | clients, dashboards |
| GET | `/v1/events?after=&limit=&wait_ms=` | pull events, long-poll | indexer |
| GET | `/v1/node` | chain id, version, height, peers | operators, clients |
| GET | `/health` | 200 when the node serves | operators, Docker, Jenkins |
| GET | `/metrics` | Prometheus text | Prometheus |

Rules for the endpoints:

- `POST /v1/txs` reads the body as borsh or JSON by `Content-Type`. A body
  that does not decode answers 400. The node decodes JSON into the same
  structure, encodes it to borsh, and checks the signature and computes the
  txid over those bytes. The JSON text itself is never hashed. It checks signature, chain id, nonce, balance, limits and circuit
  existence, then puts the tx in the mempool. The txid in the answer is the
  handle for polling.
- `GET /v1/txs/{txid}` answers 404 for a txid the node has never seen. The
  status is
  `pending` while the tx is in the mempool and `included` once a block holds
  it. For a proof request, the pending tx is the request and the included entry
  is the proof tx.
- `GET /v1/txs/{txid}/inclusion-proof` answers 404 until the tx is included.
  For a proof request the leaf is the proof tx hash.
- `GET /v1/blocks` and `GET /v1/headers` take an inclusive range. A `to` above
  the tip is cut to the tip. A range over the cap answers 422 with the code
  `range_too_large`.
- `GET /v1/headers` answers a JSON list of header views. `GET /v1/blocks`
  answers a JSON list of block views, and with `Accept: application/octet-stream`
  it answers the borsh blocks one after another, each prefixed by its length
  as a u32 (little-endian).
- `GET /v1/accounts/{addr}` answers 404 for an address the chain has not
  seen.
- `/metrics` always contains at least `pouw_node_height`,
  `pouw_node_mempool_size`, `pouw_node_peers` and
  `pouw_node_verification_duration_seconds` (histogram). The full list belongs
  to the metrics contract owned by @Faustynn.

## Flows

### A client submits a proof request and checks inclusion

1. The client reads the circuit and its sender nonce.

   `GET /v1/circuits/circ:710347f25bec15b1dce6bed71a7ab719739f0011cda6830e5eb7d3d01b000590` answers:

   ```json
   {
     "id": "circ:710347f25bec15b1dce6bed71a7ab719739f0011cda6830e5eb7d3d01b000590",
     "system": "groth16",
     "constraint_count": 20000,
     "owner": "addr:958f0654a70f1dc0ecc52081aa74391b94a31f27dba7956e6f27148b9f142c92",
     "registered_at": 3
   }
   ```

   `GET /v1/accounts/addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29` answers:

   ```json
   { "address": "addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29", "balance": 1000000, "nonce": 4 }
   ```

2. The client builds the body, signs `pouw-zk/tx-sig/v1` plus the borsh bytes
   of the body and sends the body with the signature using `POST /v1/txs` and
   `Content-Type: application/json`:

   ```json
   {
     "body": {
       "chain_id": 1,
       "sender": "addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29",
       "nonce": 4,
       "fee": 120,
       "kind": "proof_request",
       "circuit": "circ:710347f25bec15b1dce6bed71a7ab719739f0011cda6830e5eb7d3d01b000590",
       "public_inputs": ["8dcdb5739ba01795ab4e12cb8b6406369ff7169af06a04bdac5fa26bd7bc19", "3d0b7f71387c6ceb4f189d30212498dc74b4aca3faf45e44d4be3d6e1fa882"],
       "witness": "UOch5JwBPwDGLPWfIWNUKg=="
     },
     "signature": "6LwWPILu4YczKIx9SsY22zpt6wE+8tN7aDIr4g7cRcyrItY4ZGzZX0L5hUUeM9anh9WNQYsy6/mq/fPMzd0c9A=="
   }
   ```

   The client may send the borsh bytes of the SignedTx instead, with
   `Content-Type: application/octet-stream` (157 bytes or more, this
   ProofRequest is 235 bytes). The byte layout is in [block.md].

   The answer is HTTP 202:

   ```json
   { "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a" }
   ```

3. The client polls `GET /v1/txs/tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a` with `Accept: application/json`. While the tx waits:

   ```json
   { "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a", "status": "pending", "tx": { "...": "SignedTx view, with txid" } }
   ```

   After a block includes it, the answer carries the derived fields:

   ```json
   {
     "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a",
     "status": "included",
     "tx": { "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a", "body": { "...": "as in the ProofRequest view" }, "signature": "6LwWPILu4YczKIx9SsY22zpt6wE+8tN7aDIr4g7cRcyrItY4ZGzZX0L5hUUeM9anh9WNQYsy6/mq/fPMzd0c9A==" },
     "height": 12,
     "index": 3,
     "eta": "6c6f7e0023e76c6e465f8156e37ca2f42e288f36da6b10917e227b39b2287f",
     "mask": "47e9cb6a7f710d154971c9044903e699e8e8639ea18b8b6dd7e7255e9b2db6",
     "proof_system": "groth16",
     "usage_counter": 7,
     "proof": "9kVR/NbweCPLh5cc+5FEZCXaGChrOrHvk14MvXpp9oo5Rspk/3jZPKYQkKQ3y7az0soNSI9fnM8wWWCDaLJ2k0O7ANDOd5ClO5ElazcMiHskeRpVOab7+3DFhw6Mka5dq3H8TIocTWK5ICs27nwH3TmKCQejcDe9jDlZ1q9XNgg=",
     "proof_hash": "proof:26a854c7af0a340d5e77269a060fdd235f856c56c90830da9fc0aaea6f967563"
   }
   ```

4. The client fetches `GET /v1/txs/tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a/inclusion-proof`:

   ```json
   {
     "height": 12,
     "index": 3,
     "siblings": ["c114da15a58b2c2c644fb94d3aeb66502e781abc675511e6aaffaadb194f0c38", "e63d24208335aa52ae3093f0f9e1adcd9ac9ad35d3b2ef28dc1385bc64130dd6", "977f552abe7cc46da64f38ccc94601deb2a4e9c567dd25cf536464abbc949cbe"]
   }
   ```

5. The client reads `GET /v1/blocks/12/header` and compares the recomputed root
   with `header.tx_root` (see the light-client flow below for the rule).

A rejected submission answers 422, for example:

```json
{ "error": { "code": "bad_nonce", "message": "expected nonce 4, got 3" } }
```

### The indexer reads blocks and follows events

1. The indexer starts with `GET /v1/node` and reads `height`.
2. It reads blocks in ranges of at most 100. `GET /v1/blocks?from=0&to=99`
   with `Accept: application/octet-stream` answers the borsh blocks in
   ascending height order, each prefixed by its u32 length. With `Accept:
   application/json` it answers `{"blocks": [ <Block>, ... ]}` with the shape
   described in [JSON view]. It repeats with
   `from=100` and so on up to the tip.
3. It then follows the event log: `GET /v1/events?after=0&limit=100&wait_ms=20000`.
   The node holds the request up to 20 s when nothing is new. Defaults are
   `after=0`, `limit=100`, `wait_ms=0`. The maxima are 1000 and 30000, and a
   larger value is lowered to the maximum. The first `seq` is 1. The answer has
   `next`, the seq of the last returned event, or `after` when `events` is
   empty:

   ```json
   {
     "events": [
       {
         "seq": 1841,
         "time": "2026-11-02T10:15:30.123Z",
         "node_id": "node-a",
         "kind": "block_accepted",
         "data": {
           "hash": "blk:dd532958f6a2a903a20f7f7923be98a644f600624193f181ddfe31cb4f53b8b7",
           "height": 12,
           "miner": "addr:958f0654a70f1dc0ecc52081aa74391b94a31f27dba7956e6f27148b9f142c92",
           "tx_count": 1,
           "proof_count": 1,
           "bucket_count": 4,
           "bucket_id": 2
         }
       }
     ],
     "next": 1841
   }
   ```

   The indexer stores the last `seq` it processed and resumes from it after a
   restart. Kinds, their `data` and the 410 answer for an expired cursor are in
   [node-events.md]. On 410 the indexer resyncs blocks through
   `GET /v1/blocks?from=&to=` and continues events from the oldest available
   seq.
4. Blocks are the source of truth. The indexer takes balances and inclusion
   from blocks, and from events only what blocks never contain.

### A light client verifies an inclusion proof

1. It downloads headers: `GET /v1/headers?from=0&to=999` answers
   `{"headers": [ {"hash": "blk:...", "header": { ... }}, ... ]}`, at most 1000
   per call. It checks that each `prev_hash` equals the previous `hash`.
2. It asks `GET /v1/txs/tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a/inclusion-proof` and takes `index`
   and `siblings`. The leaf id is the 32 hash bytes of the txid,
   or of the `proof_hash` for a proof request (the hex after the prefix).
3. It recomputes the root with the inclusion proof algorithm of
   [block.md][block-inclusion-proof]. The verifier takes the leaf count from
   `tx_count` of the header at `height`.
4. The result must equal `tx_root` of the header at `height`. For a proof
   request, the client also checks that the `proof_hash` it got from
   `GET /v1/txs/{txid}` is the leaf id it used.

## For the workload generator

The generator signs with its own key and posts borsh. The node holds no key for
it. The generator sends a `WorkloadTask {task_id, circuit_type, circuit_size,
proof_system, fee, client_id}` ([pouw-workload PR #15],
[pouw-workload#7]). It
becomes a `POST /v1/txs` call as follows:

| WorkloadTask field | In the API |
| --- | --- |
| `circuit_type`, `circuit_size` | select a registered circuit id from the catalog. The generator keeps the table (circuit_type, circuit_size) to `circ:` id, filled when it registers the catalog circuits, and can check it with `GET /v1/circuits` (`constraint_count`) |
| `proof_system` | follows from the circuit. The generator can check it against `system` in the registry entry |
| `fee` | `body.fee` |
| `client_id` | selects the signing key. `body.sender` is its address and `body.nonce` comes from `GET /v1/accounts/{addr}` |
| `task_id` | the txid the node returns in the 202 answer |

The task has no input values. The generator builds `public_inputs` and
`witness` for the chosen circuit type.

Python has the libraries for this: `borsh-construct` for the encoding and
`pynacl` for Ed25519. The test vectors in [block.md][block-test-vectors]
give the exact bytes, the signature and the txid to check an encoder against.

## JSON view

The JSON view is for explorers, dashboards and people. A node accepts borsh
or JSON for a transaction, and both decode to the same bytes. Nothing in the
JSON text is an input to any hash or signature.

A GET of a block, header or tx returns it when the request has
`Accept: application/json`. The view adds the derived fields (ADR-0008). Fields
marked derived are not stored in the block (ADR-0007).

- A tx view has `txid` added.
- A proof tx view adds `proof_hash`, `request_txid` and the derived fields
  `eta`, `mask` and `proof_system`.
- A block view adds `hash`.

### Signed transactions

The body is one flat object. `kind` selects which other fields are present
beside the common ones (`chain_id`, `sender`, `nonce`, `fee`). Every view has
`txid`. A request body has the same shape without `txid`.

Transfer:

```json
{
  "txid": "tx:1b68b9df54d0ed19660cbf55e77eb77b9bf2eeec9d7f68ead222fdc22d893181",
  "body": {
    "chain_id": 1,
    "sender": "addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29",
    "nonce": 3,
    "fee": 10,
    "kind": "transfer",
    "to": "addr:4834fa49e799082bf2f04b378ae58014c1d2ad0237aa00ff936b99c0c89833d8",
    "amount": 500
  },
  "signature": "rTKIRqoYsyozWBY3RRHKwQY8cEuMV5meUdqfkIKQp6R/SvlecMAwHBvRxYO0dU8HcciDVoZkd31rQR7fojK14Q=="
}
```

ProofRequest (the body of the first flow, with `txid`):

```json
{
  "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a",
  "body": {
    "chain_id": 1,
    "sender": "addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29",
    "nonce": 4,
    "fee": 120,
    "kind": "proof_request",
    "circuit": "circ:710347f25bec15b1dce6bed71a7ab719739f0011cda6830e5eb7d3d01b000590",
    "public_inputs": ["8dcdb5739ba01795ab4e12cb8b6406369ff7169af06a04bdac5fa26bd7bc19", "3d0b7f71387c6ceb4f189d30212498dc74b4aca3faf45e44d4be3d6e1fa882"],
    "witness": "UOch5JwBPwDGLPWfIWNUKg=="
  },
  "signature": "6LwWPILu4YczKIx9SsY22zpt6wE+8tN7aDIr4g7cRcyrItY4ZGzZX0L5hUUeM9anh9WNQYsy6/mq/fPMzd0c9A=="
}
```

RegisterCircuit:

```json
{
  "txid": "tx:7af440a4841927cdb5352a27ba5d9e2ea541dcff0cd28d11dbf2e99d78c14df1",
  "body": {
    "chain_id": 1,
    "sender": "addr:958f0654a70f1dc0ecc52081aa74391b94a31f27dba7956e6f27148b9f142c92",
    "nonce": 0,
    "fee": 50,
    "kind": "register_circuit",
    "system": "groth16",
    "constraint_count": 20000,
    "verifying_key": "hjDGya8HMMPpY1pEyXv7SsT/V8RJlRpghIrGZvXy3gw/fA8EGX63qGEP+srsyJgMmSQK+rI4YEBctf6Lw9jl9Q==",
    "circuit": "Ln0sA6lQeuJl7PW1NWiFpTOTogKdJBOUmXJloaJa78acCr5RxuZlXYHeLQRNT7GUkx8FjAQmxnxyhdj1ZX7WSnwcl98XwGaSSCKwrwmmUlFVSWLGHiMymu0EzRkCDcO4"
  },
  "signature": "QSQrn65W+tTm533+M8sY0cP8WD+YjPJe+fLZvg1EC7u5T/lsFNzesBkenwQ+EEszq5crvOORA4edF56vR4Op/w=="
}
```

The id of the circuit is `circ:` plus the 64 hex digits of this txid.

### Proof tx

An entry of `proof_txs` holds the request, the stored usage counter and the
proof. The other fields are derived:

```json
{
  "proof_hash": "proof:26a854c7af0a340d5e77269a060fdd235f856c56c90830da9fc0aaea6f967563",
  "request_txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a",
  "request": { "txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a", "body": { "...": "ProofRequest as above" }, "signature": "6LwWPILu4YczKIx9SsY22zpt6wE+8tN7aDIr4g7cRcyrItY4ZGzZX0L5hUUeM9anh9WNQYsy6/mq/fPMzd0c9A==" },
  "usage_counter": 7,
  "proof": "9kVR/NbweCPLh5cc+5FEZCXaGChrOrHvk14MvXpp9oo5Rspk/3jZPKYQkKQ3y7az0soNSI9fnM8wWWCDaLJ2k0O7ANDOd5ClO5ElazcMiHskeRpVOab7+3DFhw6Mka5dq3H8TIocTWK5ICs27nwH3TmKCQejcDe9jDlZ1q9XNgg=",
  "eta": "6c6f7e0023e76c6e465f8156e37ca2f42e288f36da6b10917e227b39b2287f",
  "mask": "47e9cb6a7f710d154971c9044903e699e8e8639ea18b8b6dd7e7255e9b2db6",
  "proof_system": "groth16"
}
```

`eta` and `mask` are the two first public inputs of the circuit, as field
elements. `proof_system` comes from the registry entry of the circuit.
`proof_hash` is the SHA-256 of the borsh bytes of the proof tx.

### Block

```json
{
  "hash": "blk:dd532958f6a2a903a20f7f7923be98a644f600624193f181ddfe31cb4f53b8b7",
  "header": {
    "version": 1,
    "height": 12,
    "prev_hash": "blk:e70b4d067b77aece04d1ff7a1520714244d8052128a570f9fbfee81dd884748c",
    "timestamp": 1793614530,
    "miner": "addr:958f0654a70f1dc0ecc52081aa74391b94a31f27dba7956e6f27148b9f142c92",
    "difficulty": 1000,
    "bucket_count": 4,
    "bucket_id": 2,
    "tx_count": 2,
    "tx_root": "fed4199357214dfb94724176c68a76698dbc3c2d36317a895320207ac910dea6"
  },
  "txs": [ { "txid": "tx:1b68b9df54d0ed19660cbf55e77eb77b9bf2eeec9d7f68ead222fdc22d893181", "body": { "...": "Transfer" }, "signature": "rTKIRqoYsyozWBY3RRHKwQY8cEuMV5meUdqfkIKQp6R/SvlecMAwHBvRxYO0dU8HcciDVoZkd31rQR7fojK14Q==" } ],
  "proof_txs": [ { "proof_hash": "proof:26a854c7af0a340d5e77269a060fdd235f856c56c90830da9fc0aaea6f967563", "request_txid": "tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a", "...": "as in Proof tx" } ]
}
```

`txs` holds Transfer and RegisterCircuit only, never ProofRequest. In the
genesis block `prev_hash` is `blk:` followed by 64 zeros. The header endpoints
answer `{ "hash": "blk:...", "header": { ... } }`.

### Circuit entry

Default form:

```json
{
  "id": "circ:710347f25bec15b1dce6bed71a7ab719739f0011cda6830e5eb7d3d01b000590",
  "system": "groth16",
  "constraint_count": 20000,
  "owner": "addr:958f0654a70f1dc0ecc52081aa74391b94a31f27dba7956e6f27148b9f142c92",
  "registered_at": 3
}
```

With `?include=keys` the entry also has `verifying_key` and `circuit`, both
base64.

### Account

```json
{ "address": "addr:8fac0a4f4ac266a66249d7c30853b8aafefa405e9f2603b0f820100f34296e29", "balance": 1000000, "nonce": 4 }
```

### Mempool and node info

```json
{
  "tx_count": 2,
  "proof_request_count": 1,
  "txids": ["tx:4804e3ae090e3fb82de9a66aca0543c39d1ffff786133871f382ed612053687a", "tx:1b68b9df54d0ed19660cbf55e77eb77b9bf2eeec9d7f68ead222fdc22d893181"]
}
```

```json
{ "chain_id": 1, "version": "0.1.0", "height": 12, "peers": 3 }
```

## Versioning

The path prefix is `/v1`. Adding an endpoint, an optional field or an enum
value keeps the version. Removing or renaming a field or an endpoint, or
changing a meaning, bumps the prefix to `/v2`. Clients ignore unknown fields.

## Open points

- Rate limits per client and per IP: decides @IliaPoliak.
- Is the cap of 100 blocks per `GET /v1/blocks` call enough for the indexer's
  initial sync: decides @NotGeorgeLucas.

## Links

- [block.md], the block format and the hash rules.
- [node-events.md], the event log read through `/v1/events`.
- [ADR-0007], stored and derived fields.
- [ADR-0008], the node returns decoded views and derived fields.
- [pouw-workload#7], task format for the node.
- [pouw-workload PR #15], the current generator.

[Kontrakty]: index.md
[node-api.yaml]: node-api.yaml
[block.md]: block.md
[node-events.md]: node-events.md
[JSON view]: #json-view
[block-inclusion-proof]: block.md#inclusion-proof
[pouw-workload PR #15]: https://github.com/PoUW-ZK/pouw-workload/pull/15
[pouw-workload#7]: https://github.com/PoUW-ZK/pouw-workload/issues/7
[block-test-vectors]: block.md#test-vectors
[ADR-0007]: ../decisions/0007-ulozene-a-odvodene-polia.md
[ADR-0008]: ../decisions/0008-indexer-a-explorer.md
