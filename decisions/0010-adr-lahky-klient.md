---
title: "ADR-0010: Light client in the node repository"
parent: Rozhodnutia (ADR)
nav_order: 10
---

# ADR-0010: The light client is a Rust crate of pouw-node, used by the workload generator

**Status:** proposed
**Date:** 2026-10-14
**Author:** @Ddystopia

## Context

The assignment asks for "a clients, a reference node" and a workload generator
"capable of generating continuous synthetic proof-generation requests". On the
first meeting (2026-10-06) the clients were described as light nodes that act
as our test clients: they create proof requests and coin transfers and pay for
them.

A **light client** (light node) keeps no full chain. It:

- creates and signs its own transactions (proof requests, coin transfers),
- sends them to a full node, which checks them and keeps them in its mempool
  until a miner puts them into a block,
- follows block headers and checks a **Merkle proof** that its transaction is
  in a block (a short list of hashes from the transaction up to the root
  stored in the header).

It has no mempool, does not validate whole blocks and does not mine.

The current plan puts the light clients into pouw-workload (Python) next to
the generator. Every step above must produce or check bytes exactly as the node
does: transaction encoding, transaction id, signature, Merkle root and proof,
header hash. All of them follow from `contracts/block.md` and are implemented
in the node crates anyway.

## Options

1. **Light client in Python, in pouw-workload**
   - pros: the generator and the clients are in one language and one
     repository. Nobody on the workload side needs Rust.
   - cons: a second implementation of encoding, hashing, signing and Merkle
     proofs. Every change of `block.md` has to land in two code bases in two
     languages. A mismatch shows only at run time, as transactions the node
     rejects. Needs shared test vectors and review across both languages.
2. **Light client as a Rust crate and binary in pouw-node**, built from the
   node's own crates (types, crypto, network client). The generator uses it
   in one of three ways, to be chosen when the crate exists:
   - Python bindings (PyO3, packaged with maturin),
   - running the binary,
   - a WebAssembly build of the crate, loaded from Python with wasmtime-py.
     The same build runs in a browser, so the dashboard could sign and check
     inclusion proofs on the client side. Networking is not available inside
     WebAssembly by itself, so the host (Python or the browser) does the
     requests and passes the bytes in.
   - pros: one implementation of the format. A change of `block.md` changes
     both sides in one pull request and breaks the build where they disagree.
     Node CI runs the light client against a real node. Little new code,
     because the parts already exist in the node.
   - cons: the workload image needs a Rust build (a wheel from maturin, the
     binary or a `.wasm` file). The light client can start only after the
     node's types and network crates exist.

## Decision

**Proposed**, the team still has to approve it: option 2.

- The light client is a thin crate and binary inside the pouw-node codebase.
  It is the node's own types, crypto and API client plus a few hundred lines
  of glue (keys, building and signing proof requests and coin transfers,
  sending them, fetching headers and Merkle proofs, checking them). The
  skeleton already has `crates/pouw-light-client` and
  `crates/pouw-api-client`. It is trivial to add there, and whoever works on
  the node adds it.
- pouw-workload (@Quolen) keeps all client behavior: scenarios, arrival
  rates, which tasks and how many clients. It drives the light client through
  Python bindings, the binary or a WebAssembly build.

## Consequences

- One crate and one binary in pouw-node. The way the generator calls them
  (bindings, binary or WebAssembly) is chosen when the crate exists.
- Node CI runs the light client against a node.
- Order: the light client starts after `contracts/block.md` and
  `contracts/node-api.md` are agreed and the node's types and network crates
  exist. Until then the generator sends requests in the JSON of
  `node-api.md` to a stub node, without signatures.
- `contracts/node-api.md` gets two read endpoints for light clients: headers
  from a given height, and the Merkle proof of a transaction. The block header
  carries the Merkle root of its transactions.
- The node's types, crypto and network crates must not depend on the mempool,
  storage or the provers, so that the light client builds without them. CI
  builds the light-client binary on its own to keep it so.
- Checking a header in a proof-of-useful-work chain may need the winning proof
  and one proof verification, depending on how the lottery ticket is derived
  (SNARKChain Sec. IV-B). This belongs to the design of `block.md`. Until
  consensus exists, the light client trusts the node it talks to.
