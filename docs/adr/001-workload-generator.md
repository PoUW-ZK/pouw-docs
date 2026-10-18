# ADR-001: Workload Generator Architecture

- **Status:** Accepted
- **Date:** 2026-10-18

## Context

The project needs a synthetic client to stress-test the PoUW network and
simulate realistic proof demand. The generator must represent multiple task
types and arrival models, including Poisson and burst workloads, while
remaining suitable for sustained asynchronous execution.

## Decision

We will implement the workload generator in Rust with Tokio for asynchronous
performance and efficient task scheduling.

The architecture will use:

- A YAML-based `Scenario` parser for names, rates, durations, arrival models,
  and task distributions.
- A modular `NodeClient` for HTTP requests to the node API, including a mock
  mode for local development and testing.
- A `WorkloadTask` payload containing the task identifier, circuit type and
  size, proof system, fee, and client identifier.
- A separate modeling workflow in a Python Jupyter notebook before arrival
  models are fully implemented in the Rust loop.

## Consequences

Rust and Tokio provide high performance and align the generator with the core
codebase. YAML scenarios and a modular client make the tool easy to configure
and integrate with the node API.

The arrival models require manual implementation and validation. Uniform
arrival is currently supported by the execution loop; Poisson, burst, and wave
behavior will first be modeled and compared in the Python notebook before
being implemented in Rust.
