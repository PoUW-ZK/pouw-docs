# Workload Generator Analysis

This report compares three established load-testing tools and identifies
features that are useful for the custom Rust workload generator.

## Locust

Locust is a Python-based, event-driven load-testing framework. Test behavior is
written as Python user classes and tasks, while Locust distributes simulated
users and presents live execution metrics in a web UI.

Features to borrow:

- **Scriptable scenarios:** Keep scenario definitions expressive and easy to
  extend, while allowing reusable task-generation logic.
- **Distributed testing:** Support multiple generator workers coordinated by a
  controller for larger workloads.

Screenshot placeholder: `[Insert Screenshot of Locust UI here]`

## k6

k6 is a developer-focused load-testing tool with JavaScript or TypeScript test
scripts. It emphasizes automation, reproducibility, thresholds, and
integration with observability systems.

Features to borrow:

- **Scenario-specific thresholds:** Define expected rates, error limits, and
  latency targets that can fail an automated test.
- **Real-time metrics and integrations:** Export task and response metrics to
  dashboards and CI/CD systems.

Screenshot placeholder: `[Insert Screenshot of k6 Results here]`

## Apache JMeter

Apache JMeter is a GUI-assisted, Java-based load-testing tool built around
test plans, samplers, controllers, timers, assertions, and listeners. It
supports many protocols and can run non-GUI distributed tests.

Features to borrow:

- **Composable workload plans:** Represent task types, timing, assertions, and
  data sources as modular scenario components.
- **Protocol and result extensibility:** Keep the node client modular so
  additional endpoints and result formats can be added without changing the
  scheduler.

Screenshot placeholder: `[Insert Screenshot of Apache JMeter UI here]`

## Recommendations

The Rust generator should combine YAML-based scenario configuration with
scriptable task construction, explicit arrival models, and structured metrics.
Tokio provides the asynchronous runtime needed for high request rates, while a
modular HTTP client keeps node-specific behavior isolated. A future web API
can expose scenario control and metrics to a dashboard, and distributed worker
coordination can be added once single-process behavior is stable.
