# ADR-0010: Workload Generator Architecture

- **Status:** Accepted
- **Date:** 2026-10-25

## Context

We need a synthetic client to stress-test a standard Bitcoin-like L1 network
through a light node. The generator will test read operations only; it will not
submit writes or transactions. The client must generate controlled,
high-concurrency requests for chain data and provide reliable measurements of
the light node API.

## Decision

We will implement the workload generator with Java 21 and Gradle.

The architecture will use:

- `java.net.http.HttpClient` for asynchronous HTTP/2 requests to the light node
  API.
- Virtual Threads to support high concurrency with lightweight request
  execution.
- An open-loop scheduler that emits read operations according to the configured
  arrival rate instead of waiting for each previous request to finish.
- A `Semaphore` to limit the number of in-flight requests and provide
  controlled throttling.
- A `LightNodeClient` for reading the chain head, blocks, balances, and
  transaction status.
- WireMock tests against the light node API contract before the real node is
  ready.

## Consequences

Java 21 and Virtual Threads provide high concurrency and align the generator
with the new Java stack. Gradle provides a standard build and dependency
workflow, while the HTTP client and modular light-node client keep transport
and API concerns isolated.

The generator will initially depend on the light node API contract rather than
a live node. WireMock will provide deterministic contract-level tests for
chain-head, block, balance, and transaction-status reads until the real node is
ready.
