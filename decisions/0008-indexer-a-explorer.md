---
title: "ADR-0008: Indexer and explorer"
parent: Rozhodnutia (ADR)
nav_order: 8
---

# ADR-0008: Indexer and explorer, reuse or build

**Status:** proposed, open for discussion
**Date:** 2026-10-13
**Author:** @Ddystopia

This ADR is deliberately open-ended. It records a survey and a starting
proposal, not a decision. Please comment on the pull request, add options we
missed, or argue for a different proposal. The status changes only after the
discussion.

## Context

Our supervisors suggested we may not need to write our own indexer and could take an open-source one. I surveyed what exists. This ADR has the result, a proposal, and some questions for each of you. Please push back on anything here, nothing is decided.

Terms used below:

- **Indexer**: a program that reads data from the node and stores it in a database so it can be searched and queried fast.
- **Explorer**: a website that shows blocks, transactions and addresses (like blockchain.com or Etherscan).
- **RPC**: the HTTP interface of the node that other programs call to ask for data.
- **Node event**: a message the node emits when something happens (for example "proof received"), even if that thing never ends up in a block.

### Summary

- Proposal: do not adopt an existing explorer. Every surveyed explorer is built for one chain family (Bitcoin, Ethereum-like, Cosmos, Substrate) and would need its data layer replaced to read our chain.
- Much of what we want to show never appears in a block: losing proofs, buckets and losers per round, mempool, timings. No block-based explorer can show it, adapted or not.
- So the node should return decoded JSON through its RPC and emit an event stream. A thin indexer writes both into Postgres, Grafana (already deployed) reads Postgres for research panels, and pouw-dashboard is the explorer UI.
- Open-source reuse still happens, at the edges: Postgres, Grafana, optionally Hasura (GraphQL on top of Postgres). A one-time copy of a small UI as a head start is also possible.
- The frontend-plugin plus wasm decoder idea is not wrong, but it is more work than node-side JSON and no surveyed explorer could host it.

### What we want to show

Condensed from the survey and from [pouw-dashboard#13](https://github.com/PoUW-ZK/pouw-dashboard/issues/13). Grouped by where the data comes from.

**In the block** (anyone reading blocks can get it):

- Block list, height, hash, parent, time, miner, difficulty, rewards and fees.
- Coin transfers, balances, address history, search by height, hash, address, circuit id.
- Proof transactions per proof system (Groth16, PLONK, Marlin), proof sizes, fee per constraint.
- Circuit registry (id, owner, constraints, proof system).
- Winning proofs per bucket.
- From [ADR-0007](0007-ulozene-a-odvodene-polia.md): the usage counter, and the bucket count and bucket id in the header. The integrity parameter eta, the mask and the proof-system id are derived, so a tool that only mirrors raw blocks does not see them. Our node API has to return them already computed, including the eta chain (header hash, then each proof's eta) and the mask with its check.

**Node events** (never in a block, must be emitted by the node):

- Losing proofs, buckets per round, losers and near misses per round.
- Rejected proofs, unanswered client requests, orphaned blocks.
- Setup progress (pending, running, done).
- Mempool contents and pending count.
- Times: proof received, verification duration (see the timestamp question below).

**Node metrics** (Prometheus and Grafana, per node and live):

- CPU, RAM, disk, network, peer count, sync state, gossip lag, block propagation delay.
- Verification latency histograms, mempool size, admit and reject rate, prover queue depth.

**Derived by the indexer** (computed from blocks and events):

- Finality and pending/included/verified/finalized states, indexed height versus finalized height.
- Proving-time distribution, miner share of power, proofs per round, amortization indicators.
- Search by work id or proof id, pagination and filters, history.

What a block does not contain: losing proofs, buckets and losers per round, mempool, and any timing other than the block time. The progress-freeness and amortization questions in pouw-dashboard#13 cannot be answered from blocks alone.

**Answer to the open question in pouw-dashboard#13 (are timestamps needed, and who stores them):** yes, they are needed for proving-time distribution, asymmetry (prove versus verify time) and amortization metrics. A block gives only the time of inclusion. There are three sources:

1. Block time: free, fine for throughput, useless for proving duration.
2. Time claimed by the miner inside the proof transaction: on chain, but self-reported. It would be labelled "claimed", and adding fields changes the consensus format, so the node side decides.
3. Time observed by the node, emitted as events (`proof_received_at`, `verify_duration`). Different nodes see different times, so the indexer must record which node observed it, or we fix one reference node.

The indexer stores only what it is given and cannot invent timestamps. The proposal is observed times as node events (stored per proof for research), plus the same as Prometheus histograms for live panels.

## Options

### Surveyed tools

Facts read from the GitHub API on 2026-10-13. Verdicts are my judgement. I did not install or run any of these.

| Tool | Chain family it expects | License | Activity | Verdict |
|---|---|---|---|---|
| [Blockscout](https://github.com/blockscout/blockscout) | Ethereum-like (EVM) | Own license v1.0 from 2026-04-22, not open-source in the OSI sense | pushed 2026-10-13 | No. The license bars selling or hosting it commercially and distributing derivatives without a commercial license. A never-monetized student project may be inside it, but it buys nothing since our chain is not EVM. |
| [Esplora](https://github.com/Blockstream/esplora) | Bitcoin (own REST API over electrs) | MIT | pushed 2026-10-07 | No. We would have to rebuild that REST API, and the UI assumes Bitcoin-style outputs and scripts. |
| [mempool.space](https://github.com/mempool/mempool) | Bitcoin | AGPL-3.0 | pushed 2026-10-13 | No. Bitcoin-specific mempool model. |
| [BTC RPC Explorer](https://github.com/janoside/btc-rpc-explorer) | Bitcoin Core RPC | MIT | pushed 2026-02-11 | Only as a one-time copy of a small UI. The easiest to read of the group. |
| [Subscan Essentials](https://github.com/subscan-explorer/subscan-essentials) | Substrate | GPL-3.0 | pushed 2026-09-02 | No. It has a real plugin system, but plugins are server-side Go on Substrate data. |
| [Ping.pub](https://github.com/ping-pub/explorer) | Cosmos | GPL-2.0 | pushed 2026-09-28 | No. Its idea (read live from the node, no database) is worth knowing, the code is Cosmos-only. |
| [Big Dipper 2.0](https://github.com/forbole/big-dipper-2.0-cosmos) | Cosmos (GraphQL from a Cosmos indexer database) | Apache-2.0 | last push 2024-11-25, stale | No. Stale and Cosmos-only. |
| [SubQuery](https://github.com/subquery/subql), [Subsquid](https://github.com/subsquid/squid-sdk), [The Graph](https://github.com/graphprotocol/graph-node), [Ponder](https://github.com/ponder-sh/ponder) | Indexer frameworks, per-chain adapters (Polkadot, EVM, Cosmos and similar) | GPL-3.0, Apache-2.0, Apache-2.0, MIT | all active | No generic "bring your own chain" mode found. An adapter would cost more than a thin custom ingester. This was judged from READMEs. Custom-chain support in Subsquid and Envio was not verified, and Envio's license was not read. |
| [Hasura](https://github.com/hasura/graphql-engine) | Any Postgres | Apache-2.0 | active | Useful. Gives GraphQL over our tables with no backend code. |
| [Grafana](https://github.com/grafana/grafana), [Metabase](https://github.com/metabase/metabase) | Any Postgres | AGPL-3.0, AGPL-3.0 (enterprise directory under commercial license) | active | Useful. Grafana is already deployed. Good for tables and charts, weak for transaction pages with nested data. |

The search was not exhaustive, so "no generic bring-your-own-chain indexer" means I did not find one. Also, no explorer "understands RISC-V". That idea comes from zkVM projects, and our chain has no zkVM.

### Options, ranked by effort

1. **Node returns decoded JSON next to the raw bytes** (serde on the node's structs, versioned schema). Lowest effort. Any UI, curl, Python or Grafana can read it. Cost: the node team keeps the JSON and binary views in sync.
2. **Grafana dashboards over Postgres.** Low effort once rows exist. Needs option 3 first.
3. **Thin indexer plus Postgres plus a small React explorer** (the planned pouw-indexer and pouw-dashboard). Medium. We own the schema and migrations.
4. **Node team publishes a wasm decoder, explorer gets frontend plugins that render opaque data.** Medium to high. Frank assessment:
   - It pays off when there is no backend of ours, when many third parties read the chain, or when the binary format changes often and the UI should follow by bumping one package. Our audience is the team and course examiners.
   - We have a Python indexer and Postgres. The node already owns the types, so decoding in Rust and shipping JSON removes the decoder altogether. Python could call wasm, but that adds a dependency chain, so the indexer would end up asking the node for decoded data anyway.
   - None of the surveyed explorers has a documented frontend plugin API that takes opaque data for a custom chain. So we would write the frontend ourselves, which is pouw-dashboard.
   - Wasm adds a build in CI, publishing to npm and versioning against the node.
   - It becomes useful later for a "decode this hex" box or client-side proof verification. I would treat it as optional polish.
5. **Fork an existing explorer and rebase it now and then.** Highest effort. A fork pays when our changes stay at the edges, as with the Marlin library fork. Here every explorer is built around one chain family's node API and transaction model, so adapting one means replacing its data layer, which is its core. Every rebase would conflict exactly there, and upstream improvements are Bitcoin, EVM or Cosmos specific, so we would gain little from rebasing. The realistic form of reuse is a one-time copy of a small UI (BTC RPC Explorer) as a head start, never rebased.

## Decision

Proposal for discussion:

Options 1, 3 and 2, in that order:

1. Node RPC returns decoded JSON (blocks, transactions, circuits, with the derived fields from ADR-0007). The JSON is an API contract tracked in an issue, not a free-form dump.
2. Node emits an append-only event stream for what is not in blocks ([pouw-indexer#8](https://github.com/PoUW-ZK/pouw-indexer/issues/8), design of events).
3. pouw-indexer ingests both into Postgres, keeping typed columns plus the raw JSON for forward compatibility.
4. Grafana reads Postgres for research panels and Prometheus for node metrics. pouw-dashboard is the explorer, optionally over Hasura.

This answers the supervisors' suggestion through reuse of Postgres, Grafana and Hasura, not through a chain-specific explorer.

What would change it:

- The supervisors or course require a recognizable third-party explorer: then we evaluate a copy of BTC RPC Explorer and emulate its RPC. Still not recommended.
- The node cannot emit events in time: research metrics on losers and timing are lost, and we fall back to sampling the mempool through metrics.
- The node team prefers not to maintain JSON views: then the wasm decoder or a Rust-side decoding service becomes the way to keep one decoder.
- Many outside developers need to decode blocks themselves: publish the wasm decoder.
- The team shrinks: node writes rows straight to Postgres, Grafana only, no React explorer.

## Consequences

If the proposal is accepted:

- The JSON returned by the node RPC becomes a contract next to the block
  format, with the derived fields from ADR-0007.
- The node event stream ([pouw-indexer#8](https://github.com/PoUW-ZK/pouw-indexer/issues/8))
  must exist before the research panels on losing proofs and timings.
- pouw-indexer owns the Postgres schema. Grafana needs a Postgres datasource
  next to Prometheus.

## Open questions

- @NotGeorgeLucas (indexer and database): Can the indexer take "decoded JSON from RPC plus an event stream" as input and store typed rows and a raw JSON column? Do you prefer to poll the RPC or to receive pushed events?
- @v-riazantsev (web API, author of pouw-dashboard#13): With the lists above, which pouw-dashboard#13 items would you drop or defer because they need node events that may not exist yet? Is GraphQL over Hasura acceptable instead of a hand-written REST API, or do you need the versioned REST with cursors from pouw-dashboard#13?
- @ymarochok (explorer frontend): Would you start from a copy of a small existing UI (BTC RPC Explorer) or from a blank React app? Which transaction and proof pages do you need first, so we order the JSON fields by that?
- @Faustynn (team lead and DevOps): Can Grafana get a Postgres datasource next to Prometheus? Please check the list in "node metrics" and "derived" and tell me what is missing from your Grafana list.
- @IliaPoliak (node protocol, with Kostiantyn owner of the node events contract): Can the node emit the observed events (`proof_received_at`, `verify_duration`, rejected proofs, bucket closed, winner chosen)? Should the miner-claimed prove time be a proof transaction field? That is a consensus change, so it is your decision.

## Links

- [pouw-dashboard#13](https://github.com/PoUW-ZK/pouw-dashboard/issues/13): explorer and dashboard requirements.
- [ADR-0007](0007-ulozene-a-odvodene-polia.md): stored and derived fields (see "Open-source indexers and explorers").
- [ADR-0006](0006-tri-dokazovacie-systemy.md): three proof systems.

The full survey (tables with more detail and sources) is a local note. I can share it on request.
