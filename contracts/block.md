---
title: "Block and ProofTx"
parent: Kontrakty
nav_order: 1
---

# Block and ProofTx

This page fixes the byte layout and the hash rules of `Block`, `BlockHeader`,
`SignedTx`, `TxBody`, `TxKind`, `ProofTx` and the circuit registry entry.
Every component (node, light client, workload generator, indexer) reads and
writes these bytes the same way. The page is owned by @IliaPoliak and
@Ddystopia (see [index]). A change is a pull request approved by
one of them.

**Status:** proposed

**Scope:** this contract describes the semester devnet as the team builds it
(SNARKChain shape: clients submit proof requests, miners prove them, buckets,
fees). The research epic [pouw-docs#20] is expected to change the lottery
ticket rule and where eta and the mask enter the proof, the set of proving
systems (most likely one, how many is open), whether witness inputs are stored
in blocks, and the field widths. The byte-level rules (borsh, SHA-256,
Ed25519, the Merkle tree, the header shape) are expected to stay. Layout
changes come as new ADRs and a contract version bump. Read `ProofSystem`,
field widths and limits from the registry entry and the contract constants,
never hard-code them.

## Terms used below

- **borsh** is a binary serialization format with exactly one byte string for
  each value. It is used for hashing, signing, the network and the disk.
- A **txid** is the SHA-256 hash that identifies a transaction.
- The **header** is the small fixed part of a block. It is hashed and it
  commits to the rest of the block.
- A **Merkle root** is one hash that commits to an ordered list of items.
- An **inclusion proof** is the list of hashes that lets a reader check that
  one item belongs to a Merkle root without seeing the other items.
- **Public inputs** are the field elements a verifier needs to check a proof.
  They are stored in the block.
- **Witness inputs** are the bytes only the prover needs to build a proof,
  visible to every node that relays the request.
- A **field element** is a number modulo the prime order of the scalar field
  of the proof system's curve.
- The **usage counter** is the number of proofs a miner has made for one
  circuit before the current one ([ADR-0007]).
- A **bucket** is a part of the set of proof requests, selected by a prefix
  of the request hash. A miner includes only requests of one bucket
  ([ADR-0007]).

## Primitives

| Item | Rule | Reason |
| --- | --- | --- |
| Hash | SHA-256 everywhere. `Hash32 = [u8; 32]`. | One primitive to implement in every language. |
| Encoding | borsh. Integers are little-endian with fixed width. `Vec<T>` is a `u32` length and then the items. An enum is a `u8` tag and then the variant fields. A fixed array has no prefix. Field order in the structs is the wire order. | One canonical byte string per value, so hashes agree. |
| Signature | Ed25519 ([RFC 8032]). Secret key 32 B, public key 32 B, signature 64 B. Verification is strict (`verify_strict` in ed25519-dalek). | Strict verification removes signature malleability. |
| Address | The Ed25519 public key, 32 B. No hashing, no checksum. The transaction carries the sender's public key as `sender`. | The signature check needs the key, so the key is the address. |
| Chain id | `chain_id: u32` inside every signed body. Genesis fixes it. | A signature from one test network is invalid on another. |

### Field element input

`FieldInput = [u8; 31]` holds a 248-bit integer in little-endian byte order,
like every borsh integer: byte 0 is the least significant. The JSON view of
[node-api.md] shows the 62 hex digits of these bytes verbatim,
with no prefix, so no reordering happens anywhere.
The constant is `FIELD_INPUT_BYTES = 31`.

Hash-to-field is one function. Take the SHA-256 digest, keep bytes 0 to 30
and drop byte 31.

The reason: 248 bits is below the scalar field order r of BN254 (about
2^253.6) and of BLS12-381 (about 2^254.9). It is also below r of any other
pairing curve the node may add. No reduction modulo r is needed, so a
`FieldInput` maps to exactly one field element in every backend. Every prover
backend converts a `FieldInput` into its own field element and never sees a
hash.

A value that needs more bits, such as a 256-bit hash, is split by the client
into two inputs and recombined inside the circuit. The padding from 31 bytes
to the backend's field element happens inside the node and is not visible to
clients.

The rule does not cover proof systems over small fields (64-bit or 31-bit
STARK fields). [ADR-0006] excludes them.

### Domain separation

Every hash or signature over a message has a fixed prefix, so bytes of one
kind are never read as another kind.

| Constant | Value | Used in |
| --- | --- | --- |
| `DOMAIN_TX_SIG` | ASCII `pouw-zk/tx-sig/v1` | transaction signature |
| `DOMAIN_LEAF` | byte `0x00` | Merkle leaf ([RFC 6962]) |
| `DOMAIN_NODE` | byte `0x01` | Merkle inner node |
| `DOMAIN_ETA` | ASCII `pouw-zk/eta/v1` | integrity parameter eta |
| `DOMAIN_MASK` | ASCII `pouw-zk/mask/v1` | mask |

The block hash, the txid and the circuit id are plain SHA-256 over borsh
bytes with no prefix. Their borsh bytes already start with a version or a
tag byte.

## Wire layout

The layout is written in the presentation language of [RFC 8446], section 3,
with two differences that come from borsh: every integer is little-endian,
and every variable-length vector is a `uint32` item count followed by the
items, whatever the vector's ceiling. `opaque x[n]` is `n` raw bytes with no
prefix. An `enum` is one byte. A `select` on an enum appends the fields of
the chosen case. Nothing is a pointer or an offset: a reader walks the bytes
front to back.

```text
opaque Hash32[32];
opaque Address[32];                     /* Ed25519 public key */
opaque Signature[64];
opaque FieldInput[31];                  /* 248-bit little-endian integer */

struct {
    uint32     version;                 /* 1 */
    uint64     height;
    Hash32     prev_hash;               /* all zero in the genesis block */
    uint64     timestamp;               /* Unix seconds */
    Address    miner;                   /* receives the block reward */
    uint64     difficulty;              /* consensus parameter, see open points */
    uint32     bucket_count;            /* k, ADR-0007 */
    uint32     bucket_id;               /* ADR-0007 */
    uint32     tx_count;                /* Merkle leaves: txs plus proof_txs */
    Hash32     tx_root;
} BlockHeader;                          /* 136 bytes */

enum { transfer(0), proof_request(1), register_circuit(2), (255) } TxKind;
enum { groth16(0), plonk_kzg(1), marlin(2), (255) } ProofSystem;

struct {
    uint32     chain_id;
    Address    sender;                  /* signer and payer */
    uint64     nonce;                   /* txs this sender sent before this one */
    uint64     fee;                     /* paid to the miner of the including block */
    TxKind     kind;
    select (kind) {
        case transfer:
            Address    to;
            uint64     amount;
        case proof_request:
            Hash32     circuit;         /* txid of the RegisterCircuit tx */
            FieldInput public_inputs<0..MAX_PUBLIC_INPUTS>;
            opaque     witness<0..MAX_WITNESS_BYTES>;
        case register_circuit:
            ProofSystem system;
            uint64     constraint_count; /* complexity measure of ADR-0006 */
            opaque     verifying_key<0..MAX_CIRCUIT_BYTES>;
            opaque     circuit<0..MAX_CIRCUIT_BYTES>;
    };
} TxBody;

struct {
    TxBody     body;
    Signature  signature;               /* over DOMAIN_TX_SIG || body, see Signing */
} SignedTx;                             /* a transfer is 157 bytes */

struct {
    SignedTx   request;                 /* kind must be proof_request */
    uint64     usage_counter;           /* ADR-0007 */
    opaque     proof<0..MAX_PROOF_BYTES>;
} ProofTx;

struct {
    BlockHeader header;
    SignedTx   txs<0..MAX_TXS_PER_BLOCK>;        /* transfer and register_circuit only */
    ProofTx    proof_txs<0..MAX_TXS_PER_BLOCK>;
} Block;
```

The vector ceilings are the constants of [Limits], counted in items
for `public_inputs`, `txs` and `proof_txs` and in bytes for the `opaque`
vectors. `verifying_key` and `circuit` together stay within
`MAX_CIRCUIT_BYTES`.

### Block reward

The reward is credited by the state transition, as in Ethereum, where the
header names a beneficiary. When a node applies a block, it adds to the
balance of `header.miner`:

- the subsidy, the new coins the block creates, a consensus parameter (see
  open points), and
- the `fee` of every entry of `txs` and of every request in `proof_txs`.

No transaction carries the reward, so `txs` holds only what clients signed.

## Hash rules

### Identifiers

- `txid = sha256(borsh(SignedTx))`. The signature is included.
- `block_hash = sha256(borsh(BlockHeader))`.
- `circuit_id` is the `txid` of the `RegisterCircuit` tx. The circuit entry is
  created when that tx is applied.
- The proof tx hash is `sha256(borsh(ProofTx))`. Its JSON view is `proof:` plus
  64 lowercase hex digits, matching `tx:`, `blk:`, `circ:` and `addr:` of
  [node-api.md].

### Merkle tree

The leaves are in block order. First comes every tx in `txs` (leaf id is the
txid). Then comes every entry of `proof_txs` (leaf id is the proof tx hash).

- `leaf = sha256(0x00 || id)`
- `node = sha256(0x01 || left || right)`
- When a level has an odd count, the last node is promoted unchanged to the
  next level.
- A block with no leaves has the root `sha256(0x00)`, the one-byte prefix
  alone.

These rules follow [RFC 6962].

Four leaves, ids `a`, `b`, `c`, `d`:

```text
            root = H(1 || n01 || n23)
           /                         \
 n01 = H(1 || la || lb)      n23 = H(1 || lc || ld)
    /          \                /          \
 la = H(0||a)  lb = H(0||b)  lc = H(0||c)  ld = H(0||d)
```

Three leaves, ids `a`, `b`, `c`. The leaf `lc` moves up unchanged:

```text
        root = H(1 || n01 || lc)
       /                        \
 n01 = H(1 || la || lb)          lc = H(0||c)
    /          \
 la = H(0||a)  lb = H(0||b)
```

### Inclusion proof

An inclusion proof has three parts: `height` (the block), `index` (the leaf
position) and `siblings` (the sibling hashes from the leaf up). The verifier
takes the leaf count `n` from `tx_count` of the header at `height`.

The verifier recomputes the root with this algorithm and compares it with
`tx_root` of that header.

```text
h = sha256(0x00 || id);  i = index;  n = header.tx_count
while n > 1:
    if i == n - 1 and n is odd:
        h stays                       (promoted, no sibling consumed)
    else:
        s = next sibling
        h = sha256(0x01 || h || s)    if i is even
        h = sha256(0x01 || s || h)    if i is odd
    i = floor(i / 2);  n = ceil(n / 2)
h must equal tx_root, and all siblings must be consumed.
```

### Bucket

`bucket(txid)` is the first 4 bytes of the request txid read as a
little-endian `u32`, modulo `header.bucket_count`. Every request in
`proof_txs` must satisfy `bucket(txid) == header.bucket_id`. This follows
[ADR-0007]: the bucket is
chosen by a prefix of the request hash.

### Eta, mask and public inputs

Eta is the integrity parameter ([ADR-0007],
SNARKChain Sec. IV-C). It is derived, not stored. `txs_root` is the Merkle
root over `txs` alone, built with the rules above.

```text
eta_0 = sha256(DOMAIN_ETA || prev_hash || txs_root)
eta_i = sha256(DOMAIN_ETA || eta_{i-1} || sha256(proof_{i-1}))     for i >= 1
```

The index `i` counts proof txs in the block, from 0. `proof_{i-1}` is the
`proof` bytes of the previous proof tx. The circuit receives
`hash_to_field(eta_i)`.

The mask is derived as well. `usage_counter_le_u64` is the counter as 8
little-endian bytes.

```text
mask = sha256(DOMAIN_MASK || miner || circuit_id || usage_counter_le_u64)
```

The circuit receives `hash_to_field(mask)`.

The public inputs of every circuit are laid out as:

```text
[eta, mask, client public_inputs...]
```

Every registered circuit must expose these two values as its first two public
inputs. The verifier builds the list from the block and from the
`public_inputs` of the request.

## Signing

A client signs `DOMAIN_TX_SIG || borsh(body)` with the Ed25519 secret key of
`body.sender`. Clients (the workload generator's light clients) sign
`Transfer`, `ProofRequest` and `RegisterCircuit` bodies. Their balance pays
the fees. Genesis allocates coins to the test clients' keys.

Blocks are not signed. A miner holds a key only to receive rewards, because
`header.miner` is a public key. The miner proves its work with the proofs in
`proof_txs`, and the lottery decides whether the block is valid. A block
signature would add nothing. A full node that does not mine holds no keys.

Signatures exist although the work is a proof: a proof shows that a miner did
work, and a signature shows that a client agreed to pay a fee and to spend
coins. Without signatures anyone could submit requests charged to any
account.

## Limits

| Constant | Value | Reason |
| --- | --- | --- |
| MAX_PUBLIC_INPUTS | 64 | field elements the verifier reads from the block |
| MAX_WITNESS_BYTES | 1 MiB | see [ADR-0011] |
| MAX_PROOF_BYTES | 4096 | proofs of ADR-0006 are at most 1008 B |
| MAX_CIRCUIT_BYTES | 64 MiB | a 200k-constraint R1CS plus the verifying key |
| MAX_BLOCK_BYTES | 8 MiB | borsh size of the whole `Block` |
| MAX_TXS_PER_BLOCK | 1024 | `txs` plus `proof_txs` |

Fee rule for proof requests: the fee is priced per public input element and
per witness kilobyte. The two coefficients are consensus parameters (see open
points).

## Validity rules

A node accepts a tx (into the mempool or into a block) only if all of these
hold.

1. The signature verifies strictly over `DOMAIN_TX_SIG || borsh(body)` with
   `sender` as the key.
2. `chain_id` equals the chain id of the node.
3. `nonce` equals the nonce of the sender's account.
4. The balance of the sender covers `amount` (for a `Transfer`) plus `fee`.
5. For a `ProofRequest`, `circuit` exists in the registry.
6. Every length is within the limits above. `verifying_key` and `circuit`
   together stay within `MAX_CIRCUIT_BYTES`.

A node accepts a block only if all of these hold.

1. `header.version` is 1, `height` is the parent's height plus one, and
   `prev_hash` is the hash of the parent header.
2. Every tx in `txs` is valid, is a `Transfer` or a `RegisterCircuit`, and is
   applied in order, so nonces and balances follow from the previous tx.
3. For every proof tx, the request is a valid `ProofRequest` and its circuit
   exists.
4. For every proof tx, `usage_counter` equals the previous counter of
   (`header.miner`, circuit) plus one.
5. The proof verifies against `[eta_i, mask, public_inputs...]`.
6. Every request id of `proof_txs` falls in the bucket `header.bucket_id` of
   `header.bucket_count`.
7. `tx_count` equals the number of entries of `txs` plus `proof_txs`, and
   `tx_root` recomputes from `txs` and `proof_txs`.
8. The block is within `MAX_BLOCK_BYTES` and `MAX_TXS_PER_BLOCK`.
9. The lottery condition of the consensus holds (see open points).

## Witness inputs

Witness inputs are bytes only the prover needs, but they are visible to every
node that relays the request. They are called "witness" only because the
prover alone needs them, they are not hidden. The decision to keep them in the
tx and in the block is in [ADR-0011].

## Test vectors

Every value below is reproducible in any language from the rules on this
page. The secret key is the Ed25519 seed of 32 bytes of `0x01`. The
`pouw-types` test suite pins the same values.

Public key of that seed, the sender address:

```text
8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c
```

One `TxBody` of kind `Transfer`: `chain_id` 1, `sender` the key above,
`nonce` 0, `fee` 10, `to` 32 bytes of `0x22`, `amount` 1000 (`0x03e8`).
The borsh bytes, 93 in total:

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | `01000000` | chain_id = 1, u32 little-endian |
| 4 | `8a88e3dd...b40f6f5c` | sender, 32 bytes |
| 36 | `0000000000000000` | nonce = 0, u64 |
| 44 | `0a00000000000000` | fee = 10, u64 |
| 52 | `00` | tag of `Transfer` |
| 53 | `22` repeated 32 times | to |
| 85 | `e803000000000000` | amount = 1000, u64 |

As one hex string:

```text
010000008a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c00000000000000000a00000000000000002222222222222222222222222222222222222222222222222222222222222222e803000000000000
```

Signature over `DOMAIN_TX_SIG || body bytes`, 64 bytes:

```text
507d5e19dca04f6961f00fd6f6cb0e31dffa042c43357bada9160696778aa4f45c720c67845a2251dfcd4e0fd078790dec9cd32392b651e1ecb24128960f5304
```

txid, the SHA-256 of the 157 bytes of body then signature:

```text
41ef667fcdab0af9fbf57805b90f90a60825c43d4a796b1f26980207e622e6f7
```

Merkle roots where the leaf ids are 32 bytes of `0x01`, `0x02`, `0x03` and
`0x04`, the ids themselves and not hashes of them:

| Leaves | Root |
| --- | --- |
| none | `6e340b9cffb37a989ca544e6bb780a2c78901d3fb33738768511a30617afa01d` |
| first 1 | `dcffe786ded16d283c663846ad0c4ff26558fccde36ca9d30b2ea19eade9fc0e` |
| first 2 | `3a066e0f40c6a1981ebfa60d2411625d0517ae22c2fc8c7c1784ff8a75c78565` |
| first 3 | `df896896c799531f1fd1e556cea26a6989ab06853bcbfdd3e4f5097a611f658f` |
| all 4 | `3b3c0ce45d11517a54300a196b61497c4165150d72b7782a4548e3984da771b2` |

Inclusion proof of index 2 in the tree of 4 leaves, siblings in order from the
leaf up:

```text
1da033bf8927ed69376d91533748494f7f5e88c20603dede2afc9bfd43d46f17
3a066e0f40c6a1981ebfa60d2411625d0517ae22c2fc8c7c1784ff8a75c78565
```

Inclusion proof of index 2 in the tree of 3 leaves, siblings in order from the
leaf up. The leaf is promoted at the first level, so there is one sibling:

```text
3a066e0f40c6a1981ebfa60d2411625d0517ae22c2fc8c7c1784ff8a75c78565
```

`hash_to_field` of `sha256("abc")`, in the JSON view of
[node-api.md]:

```text
ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015
```

## Open points

| Point | Decides |
| --- | --- |
| Meaning of `difficulty` and the subsidy amount | @IliaPoliak, [pouw-node#8] |
| Fee coefficients (per public input element and per witness kilobyte) | @IliaPoliak and @Ddystopia |
| Whether the eta chaining order equals the lottery order. The encoding above is fixed. | @IliaPoliak |
| The rule for k, the number of buckets | [pouw-docs#11] |
| Timestamp bounds (how far from the parent's and from the node clock a header timestamp may be) | @IliaPoliak |
| Width of `FieldInput` once the set of proving systems is settled. 31 bytes fits every 254-bit curve; a chain over a small field (Goldilocks, Mersenne-31, BabyBear) needs its own width and hash-to-field rule. | @Ddystopia, [pouw-docs#22] |

## Links

- [ADR-0006: three proof systems]
- [ADR-0007: stored and derived fields]
- [ADR-0011: inputs in the block]
- [pouw-node#9: data structures of Block and ProofTx]
- SNARKChain paper: <https://arxiv.org/abs/2510.09729>
- borsh: <https://borsh.io>
- RFC 6962 (Merkle trees): <https://www.rfc-editor.org/rfc/rfc6962>
- RFC 8032 (Ed25519): <https://www.rfc-editor.org/rfc/rfc8032>

[index]: index.md
[ADR-0007]: ../decisions/0007-ulozene-a-odvodene-polia.md
[RFC 8032]: https://www.rfc-editor.org/rfc/rfc8032
[RFC 8446]: https://www.rfc-editor.org/rfc/rfc8446
[Limits]: #limits
[node-api.md]: node-api.md
[ADR-0006]: ../decisions/0006-tri-dokazovacie-systemy.md
[RFC 6962]: https://www.rfc-editor.org/rfc/rfc6962
[ADR-0011]: ../decisions/0011-adr-vstupy-v-bloku.md
[pouw-node#8]: https://github.com/PoUW-ZK/pouw-node/issues/8
[pouw-docs#11]: https://github.com/PoUW-ZK/pouw-docs/issues/11
[pouw-docs#22]: https://github.com/PoUW-ZK/pouw-docs/issues/22
[ADR-0006: three proof systems]: ../decisions/0006-tri-dokazovacie-systemy.md
[ADR-0007: stored and derived fields]: ../decisions/0007-ulozene-a-odvodene-polia.md
[ADR-0011: inputs in the block]: ../decisions/0011-adr-vstupy-v-bloku.md
[pouw-node#9: data structures of Block and ProofTx]: https://github.com/PoUW-ZK/pouw-node/issues/9
[pouw-docs#20]: https://github.com/PoUW-ZK/pouw-docs/issues/20
