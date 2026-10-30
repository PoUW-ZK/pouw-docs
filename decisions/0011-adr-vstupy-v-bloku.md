---
title: "ADR-0011: Witness inputs in the block"
parent: Rozhodnutia (ADR)
nav_order: 11
---

# ADR-0011: Witness inputs travel inside the proof request and are stored in the block

**Status:** proposed
**Date:** 2026-10-19
**Author:** @Ddystopia

This ADR describes the semester devnet as the team builds it (SNARKChain
shape: clients submit proof requests, miners prove them, buckets, fees). The
research epic [pouw-docs#20] is expected to change whether witness inputs are
stored in blocks, the proving systems and the lottery ticket rule. Such
changes come as new ADRs and a contract version bump.

## Context

A proof request asks the network to prove a statement about a registered
circuit. The prover needs two kinds of input. Terms used below:

- **Public inputs** are the field elements that the verifier needs together
  with the proof. They are stored in the block.
- **Witness inputs** are the bytes that only the prover needs to build the
  proof, for example the pixels of a photograph or the weights of a model. The
  circuit turns them into the full assignment of wires (the witness of the
  proof system).
- A **mempool** is the list of requests that a node has accepted and that wait
  for a miner.
- **Gossip** is the relay of a message from each node to its peers.
- A **txid** is the hash of the signed transaction bytes. A **Merkle
  inclusion proof** shows with a few hashes that a transaction is part of a
  block.

Witness inputs are **not private**. Every node that relays the request sees
them, and the block shows them to everybody. We avoid the word "private" on
purpose. The SNARKChain paper ([arXiv 2510.09729],
Sec. IV-E and Appendix B to G) uses "private inputs" for inputs that are hidden
from the miner by encryption. That is a different feature, and we do not build
it.

The paper does not say where witness inputs live. It assumes that miners
"had already cached the data required to generate a proof from the circuit
registry subnetwork" (Sec. V), so the data comes from a storage network that
is not described.

### Data path of a proof request

1. The client signs the request.
2. The node checks the signature, the nonce and the fee, and keeps the request
   in its mempool.
3. The node gossips the request to its peers.
4. A miner picks the request and proves it.
5. The proof transaction enters a block.
6. Every node verifies the block.

Step 4 needs the witness. Step 6 needs the proof, the verifying key and the
public inputs, and nothing else: a verifier never reads the witness. The
mempool and the gossip carry the witness in every option below, because the
miner must have it. The options differ in what the block keeps after step 4.

### How existing chains handle large payloads

- **Ethereum calldata** is a field of the transaction. It is signed, hashed
  into the transaction hash and stored in the block forever
  ([execution-specs, transactions.py]).
- **Ethereum blobs** ([EIP-4844])
  travel next to the transaction. The block keeps only a commitment (a short
  hash of the blob) for each blob, and nodes must keep the blob for
  `MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS` epochs, "around 18 days", and may
  prune it afterwards. The commitment stays in the block.
- **Bitcoin prune mode** deletes the oldest block files and keeps the headers
  and the current state ([release notes 0.11.0]).
  It prunes whole blocks, never single fields. A pruned node can no longer
  serve old blocks.
- **Separate data layers** keep the data outside the chain that orders the
  transactions. [Celestia]
  is a blockchain that only orders and stores data for other chains.
  [IPFS] stores files under
  their content hash, and a chain keeps only the hash.

### Size of the witness

The R1CS circuits of [ADR-0006] have 10k to
200k constraints. A wire value is one field element of 32 bytes.

- Upper bound for 200k constraints: 200,000 \* 32 B = 6,400,000 B = 6.4 MB.
- Upper bound for 10k constraints: 10,000 \* 32 B = 320,000 B = 320 KB.

These are upper bounds, because a typical circuit has far fewer wires than
constraints. The witness inputs are also smaller than the wire values, because
the circuit computes the other wires from the inputs. We expect kilobytes to a
few megabytes for the catalog circuits.

Storage of a test network that makes one block per 10 s for 24 hours, with 10
proof transactions per block and 100 KB of witness each:

- Blocks: 24 \* 3600 s / 10 s = 8,640.
- Requests: 8,640 \* 10 = 86,400.
- Witness bytes: 86,400 \* 100,000 B = 8,640,000,000 B = 8.64 GB, or 1 MB per
  block (100 KB/s of traffic per node).
- At the caps below (8 MiB per block): 8,640 \* 8 MiB = 69,120 MiB = 67.5 GiB.

## Options

1. **Witness in the transaction and in the block.** The ProofRequest body holds
   `witness: Vec<u8>`.
   - pro: one data path. The mempool, the gossip and the block carry the same
     bytes.
   - pro: the input stays available from the block, for re-proving on
     another system or for looking at what was proved.
   - pro: no new component, no new failure mode, no requirement on the
     circuits.
   - con: disk and bandwidth grow with the witness size for as long as the
     block exists.
   - con: a single field cannot be pruned. The witness is inside the signed and
     hashed bytes, so removing it changes the txid and breaks the Merkle
     inclusion proofs of the transactions in the block. Whole old blocks can
     still be pruned, as in Bitcoin.
2. **Commitment in the block, witness beside the transaction.** The circuit
   hashes its witness inputs and exposes the hash as a public input. The block
   keeps the hash, the proof binds the witness to it, and the bytes travel in a
   separate message to the miners, who drop them after inclusion. A hash as a
   plain transaction field outside the circuit would not bind the proof to the
   input, so the hash has to be computed in the circuit.
   - pro: the block stays small and verifiable forever with no other data.
   - pro: the witness is pruned by default, so no node keeps old inputs.
   - con: every registered circuit must hash its inputs. SHA-256 in R1CS costs
     on the order of 30k constraints per 64-byte message block (circomlib),
     so hashing a 1 KB input adds about 480k constraints, more than the
     largest catalog circuit of ADR-0006 has in total. An arithmetic hash
     such as Poseidon costs a few hundred constraints per message block but
     ties the catalog to one hash.
   - con: a second message type, a second store with its own cap and a
     "witness missing" failure mode in the node and the miner, before proving.
   - con: the chain does not keep the input. The hash and the proof show that
     an input with that hash satisfies the circuit, which is enough for
     correctness. Whoever needs the bytes later, to re-prove on another
     system or to look at what was proved, must get them from the client.
   - con: one public input is used by the hash and `MAX_PUBLIC_INPUTS` shrinks
     by one.
3. **Witness off-chain entirely.** The request carries a hash and a location.
   This is the assumption of the paper.
   - pro: the chain carries almost no witness bytes.
   - pro: very large inputs are possible.
   - con: a storage network that we do not have, plus the availability of the
     location when a miner needs it.
   - con: the request can become unprovable after the location disappears.
   - con: the same binding question as in option 2, so the circuit hashes its
     inputs here too.

| | 1. In the block | 2. Commitment | 3. Off-chain |
| --- | --- | --- | --- |
| Block verifiable from its own bytes | yes | yes | yes |
| Data paths | 1 | 2 (tx and witness) | 2 (tx and external store) |
| Components to write | none | witness message and store | storage network |
| Requirement on circuits | none | hash the inputs in the circuit | hash the inputs in the circuit |
| Input available later | from the block | from the client only | while the location lives |
| Pruning | whole old blocks | by default | external |
| Bandwidth cost | witness in every block transfer | witness once, to the miners | witness only to the miner |

## Decision

**Proposed**, the team still has to approve it:

- **Option 1.** The ProofRequest body holds `witness: Vec<u8>`. The witness is
  inside the signed bytes, inside the block and inside the txid.
- **Caps.** `MAX_WITNESS_BYTES` is 1 MiB per request and `MAX_BLOCK_BYTES` is
  8 MiB per block. A node rejects a request or a block above the cap.
- **Shared data goes into the circuit registration.** Data that many requests
  use, such as model weights or large tables, belongs in the circuit
  registration (`RegisterCircuit.circuit`), once, and not in every request.

Why option 1 and not option 2, which is sound:

- This is a student project, and the test network runs a few nodes for hours,
  not years. The catalog circuits need kilobytes to a few megabytes of
  witness, so no disk fills, and the saving of option 2 is disk that nobody
  misses.
- Option 2 puts a hash into every catalog circuit and a second data path into
  every node. Every saved component is one that nobody has to write or debug
  across six repositories.
- A client that wants option 2 for its own circuit can still have it under
  option 1: hash the inputs in the circuit and send a short witness.
- The caps make option 1 safe. Without `MAX_BLOCK_BYTES` the workload
  generator itself could stall the network by accident.

What would change this decision:

- a deployment that must run for months,
- witnesses above a few MB in the catalog,
- a requirement that nodes do not keep client inputs.

## Consequences

- `block.md` in [Kontrakty] gets the field
  `witness: Vec<u8>` in the ProofRequest body and the two caps
  `MAX_WITNESS_BYTES` and `MAX_BLOCK_BYTES`.
- The node API returns the witness as base64.
- The indexer may skip storing the witness bytes. It can keep the length and
  the hash.
- The fee formula prices the witness bytes. This is open, see
  [pouw-node#8].
- The circuit registry stores the shared circuit blob once.
- A circuit whose witness is above 1 MiB cannot be proved on the network. The
  upper bound of 6.4 MB for 200k constraints is above the cap, and the cap is
  a constant that the team can raise.
- A node that must save disk prunes whole old blocks. Other nodes can still
  serve them.

[arXiv 2510.09729]: https://arxiv.org/abs/2510.09729
[execution-specs, transactions.py]: https://raw.githubusercontent.com/ethereum/execution-specs/mainnet/src/ethereum/forks/prague/transactions.py
[EIP-4844]: https://raw.githubusercontent.com/ethereum/EIPs/master/EIPS/eip-4844.md
[release notes 0.11.0]: https://raw.githubusercontent.com/bitcoin/bitcoin/master/doc/release-notes/release-notes-0.11.0.md
[Celestia]: https://docs.celestia.org/learn/how-celestia-works/data-availability-layer
[IPFS]: https://docs.ipfs.tech/concepts/what-is-ipfs/
[ADR-0006]: 0006-tri-dokazovacie-systemy.md
[Kontrakty]: ../contracts/index.md
[pouw-node#8]: https://github.com/PoUW-ZK/pouw-node/issues/8
[pouw-docs#20]: https://github.com/PoUW-ZK/pouw-docs/issues/20
