---
title: "ADR-0007: Stored and derived fields"
parent: Rozhodnutia (ADR)
nav_order: 7
---

# ADR-0007: Stored and derived fields of Block and ProofTx

**Status:** proposed
**Date:** 2026-10-13
**Author:** @Ddystopia

## Context

[pouw-node#9](https://github.com/PoUW-ZK/pouw-node/issues/9) lists the fields
of a Block (previous block hash, height, time, miner, difficulty, list of
proofs) and of a ProofTx (circuit, inputs, fee, client, proof). The SNARKChain
protocol ([arXiv 2510.09729](https://arxiv.org/abs/2510.09729)) uses four more
values:

- **Bucket** (Sec. IV-B5). Proof transactions are split into buckets "based
  on the prefix of their hash", and a miner includes only transactions of one
  bucket. The number of buckets is adjusted to the number of pending proof
  transactions, and Sec. V-3 says it is "constrained by the number of proofs
  that are available in the mempool".
- **Integrity parameter eta** (Sec. IV-C). A hash of the block header (the
  previous block hash and the coin transactions), given to every proof as a
  public input, so a proof cannot be moved into another block. After a lost
  lottery it is recomputed to include the new proof, which chains the proofs.
- **Mask** (Sec. IV-C). "The mask is computed as a combination of the miner's
  address and the number of times the particular circuit was utilized by the
  miner." It makes every proof input unique, so work cannot be amortized.
- **Proof-system id**: Groth16, PLONK or Marlin
  ([ADR-0006](0006-tri-dokazovacie-systemy.md)).

For each value we must decide whether the block stores it or every reader
derives it. Terms used below:

- The **header** is the small fixed part of a block that is hashed and that
  commits to the rest of the block.
- A **txid** is the identifier of a Bitcoin transaction.
- The **nonce** of an Ethereum account is the number of transactions it has
  sent so far.
- A **light client** downloads only block headers and asks full nodes for the
  rest when it needs it.
- A **state root** is one hash in an Ethereum header that commits to the
  balances and other state of all accounts after the block.

Existing chains answer the same question in four ways:

1. **Do not store what anyone can cheaply recompute from the block.** A
   Bitcoin transaction is "SHA256(SHA256()) hashed to create the TXID"
   ([Bitcoin developer reference, transactions](https://developer.bitcoin.org/reference/transactions.html)),
   and the txid is not a field of the transaction. An Ethereum transaction
   holds the signature (`v`, `r`, `s`) and no sender field, the sender is
   recovered from the signature by `recover_sender`
   ([execution-specs, transactions.py](https://raw.githubusercontent.com/ethereum/execution-specs/mainnet/src/ethereum/forks/prague/transactions.py)).
   A stored copy would be a second source of truth that every node has to
   check against the recomputed value anyway, and it costs bytes.
2. **Store what the signer must commit to, even if it could be counted from
   history.** The Ethereum nonce is a transaction field ("equal to the number
   of transactions sent by the sender", same file), and validators reject a
   transaction whose nonce differs from the account's nonce
   (`NonceMismatchError` in
   [fork.py](https://raw.githubusercontent.com/ethereum/execution-specs/mainnet/src/ethereum/forks/prague/fork.py)).
   The transaction is self-contained, has a fixed order and cannot be
   replayed.
3. **Store values that come from local or changing state, and commit derived
   state in the header where light clients need it.** A Bitcoin header stores
   `nBits`, "an encoded version of the target threshold"
   ([Bitcoin developer reference, block chain](https://developer.bitcoin.org/reference/block_chain.html)).
   Full nodes could compute it from history and they do, rejecting a header
   whose `nBits` differs from `GetNextWorkRequired` with `bad-diffbits`
   ([validation.cpp](https://raw.githubusercontent.com/bitcoin/bitcoin/master/src/validation.cpp)).
   A client that holds only headers
   ([SPV](https://developer.bitcoin.org/devguide/operating_modes.html)) can
   still check the proof of work. Ethereum headers carry `state_root` and
   `receipt_root`, commitments to data derived by executing the block
   ([blocks.py](https://raw.githubusercontent.com/ethereum/execution-specs/mainnet/src/ethereum/forks/prague/blocks.py)).
4. **Derived values belong in the node API.** `eth_getTransactionByHash`
   ([execution-apis](https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/transaction.yaml))
   returns a `TransactionInfo` with the required fields `hash` and `from`
   ([schema](https://raw.githubusercontent.com/ethereum/execution-apis/main/src/schemas/transaction.yaml)),
   neither of which is stored in the transaction.

A type tag is a related case. [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)
puts a `TransactionType` byte in front of every typed transaction because the
parser needs it before it can decode the rest.

## Options

1. **Integrity parameter**
   - store: an off-chain verifier reads it directly. But every node must
     recompute it anyway to verify the proof, so it is a second source of
     truth (principle 1).
   - derive: no extra bytes and no mismatch to check. Off-chain verifiers
     depend on the node API.
2. **Mask**
   - store the mask: readable directly, but a duplicate of the miner address
     and the counter that validators recompute anyway.
   - derive the mask and the counter from history: no bytes, but a proof
     cannot be checked without the whole history, and the miner never commits
     to the value it used.
   - store only the counter, derive the mask: a few bytes, the proof entry is
     self-contained (principle 2).
3. **Bucket**
   - derive the count and the id: no bytes, but the count depends on the
     mempool, which every node sees differently, so nodes would disagree.
   - store the count and the id in the header: a few bytes, nodes agree on
     the count, readers group blocks without reading the transactions.
4. **Proof-system id**
   - per transaction: the parser knows the system without a lookup, but it
     repeats a fact already fixed by the circuit, and validators must check
     that the two agree.
   - in the circuit registry entry: no bytes per transaction, one lookup that
     the verifier needs anyway for the verification key.

## Decision

**Proposed**, the team still has to approve it:

- **Integrity parameter: derived.** Every verifier recomputes it from data in
  the block, because it needs it as a public input to verify the proof.
  Off-chain verifiers get it from the node API together with the proof.
- **Mask: derived** from the block's miner address and a **usage counter**
  stored in each proof entry, a small integer like the Ethereum nonce.
  Validators check that the counter equals the previous count for this miner
  and circuit plus one. Anyone can compute the mask from the block alone.
- **Bucket: the bucket count k and the bucket id are stored in the header.**
  k depends on the mempool size, which every node sees differently, so it
  cannot be derived and must be agreed through the header, like `nBits`. The
  bucket id follows from the transaction hashes once k is known, but it costs
  a few bytes and lets explorers and fork choice group blocks without reading
  transactions. Validators check that every proof transaction matches it.
- **Proof-system id: in the circuit registry entry**, not per transaction. A
  circuit is registered for one proving system, so the circuit hash
  determines it, and the registry lookup is cheap. A per-proof type tag would
  only be needed if the parser had to know the system before any lookup, and
  the verifier looks up the verification key first anyway.

## Consequences

- pouw-node#9 adds to the header the bucket count and the bucket id, and to
  each proof entry the usage counter. Block and ProofTx get no field for the
  integrity parameter, the mask or the proof-system id.
- The node keeps a counter per (miner, circuit) as state, updated by every
  accepted block, in the same way Ethereum keeps the nonce per account.
- The byte encodings of these fields, of the bucket id and of the mask
  formula go to `block.md`, listed in [Kontrakty](../contracts/index.md).
- Open point: the rule that decides k is missing from the paper. Until it is
  agreed, validators cannot check the k in a header. See the issue on buckets
  and the 51% problem ([#11](https://github.com/PoUW-ZK/pouw-docs/issues/11)).
- Open point: if one miner mines in two buckets in parallel, both blocks may
  use the same next counter value for the same circuit, and the order of the
  counter is then unclear.

### Open-source indexers and explorers

A survey of open-source explorers and indexers (2026-10-13) found that every
surveyed explorer is tied to the node API of one chain family (for example
Esplora and BTC RPC Explorer to Bitcoin, Blockscout to EVM chains, Ping.pub to
Cosmos, Subscan to Substrate), and none decodes custom chain data by itself.

- The derived values (integrity parameter, mask, bucket id from the hashes,
  proof-system id through the registry) are not in the raw block. A generic
  indexer that reads only raw blocks cannot show them without re-implementing
  our hash rules and keeping registry state.
- The node API and the node event stream therefore return them already
  computed (principle 4, as Ethereum RPC does). This also suits Grafana and an
  indexer based on Postgres. A tool that only mirrors raw blocks, for example
  a plain block dump into Postgres, would miss them.
- The stored counter and the header fields k and bucket id are visible to any
  tool that parses the block.
