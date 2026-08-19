<a href="https://supra.com">
 <img width="100%" src="./.assets/supra_banner.png" alt="Supra Banner" />
</a>

---

[![Discord chat](https://img.shields.io/discord/850682587273625661?style=flat-square)](https://discord.gg/supralabs)

[Supra](https://supra.com) is a layer one blockchain with a focus on vertical integration and low latency.

This repository is the canonical platform for publishing releases of Supra's Validator and RPC node binaries. The code is currently closed-source, and will remain so while the implementation matures. Stable versions will be published in this repository when auditing is complete.

The [Supra Move Framework](https://github.com/Entropy-Foundation/aptos-core/tree/dev/aptos-move/framework/supra-framework) and the [Supra AptosVM](https://github.com/Entropy-Foundation/aptos-core/blob/dev/aptos-move/aptos-vm/src/aptos_vm.rs) can be found in [our fork of aptos-core](https://github.com/Entropy-Foundation/aptos-core).


## Documentation

* [Mainnet status](./docs/release/mainnet/README.md) — the versions and feature flags live on
  mainnet today
* [Release changelogs](./docs/release) — what changed in each mainnet release
* [Node configuration](./docs/operations/node-configuration) — every field in
  `smr_settings.toml`, `genesis_parameters.toml`, and the RPC node's `config.toml`,
  with mainnet templates
* [Observability](./docs/operations/observability) — collecting metrics, logs, and
  traces from your nodes, plus a [ready-to-run Grafana stack](./observability)
  with Supra dashboards and alert rules

## Getting Started

* [Supra](https://supra.com)
* [Supra Developer Docs](https://docs.supra.com/move/getting-started)
* Follow us on [Twitter](https://twitter.com/SUPRA_Labs).
* Join us on the [Supra Discord](https://discord.gg/supralabs).
