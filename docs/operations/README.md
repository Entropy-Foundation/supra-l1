# Operations

Guides for configuring and monitoring Supra validator and RPC nodes.

## Node configuration

Reference for every field in the node configuration files, with annotated
mainnet templates alongside each guide.

- [Validator (`supra`)](./node-configuration/supra/config.md) —
  `smr_settings.toml` and `genesis_parameters.toml`
- [RPC node (`rpc_node`)](./node-configuration/rpc_node/config.md) — `config.toml`

> These guides describe the **current** release. For the configuration as it
> stood at an earlier release, check out this repository at that release's tag.
> Changes between releases are summarised in the
> [release changelogs](../release/mainnet).

## Observability

How to collect metrics, logs, and traces from your nodes, and what to do with
them once you have.

- [Overview](./observability/README.md) — what a node emits and how it leaves
- [Collecting telemetry](./observability/node-telemetry.md) — ports, scraping, tracing
- [Running the bundled stack](./observability/running-the-stack.md) — Grafana in one command
- [Reading the dashboards](./observability/dashboards.md) — including peer liveness
- [DKG observability](./observability/dkg.md) — diagnosing a stuck epoch boundary
- [Profiling](./observability/profiling.md) — investigating memory growth

The dashboards, alert rules, and collector configuration themselves live in
[`observability/`](../../observability) at the repository root.
