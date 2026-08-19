# Release changelogs

User-facing changes for each release, grouped by the surface they affect: the
CLIs, the REST API, the node configuration files, the Supra Move framework, and
on-chain feature activation.

Releases are recorded **per network**, because the two do not advance in step: a
framework version is often rolled out at a different binary version on testnet
than on mainnet, and some releases are superseded during testing and never reach
mainnet at all. Each network directory therefore carries its own chain.

## Mainnet

> **[What mainnet is running right now](./mainnet/README.md)** — live framework version, the
> feature flags that are active and since when, and what that means if you run a validator.

| Release | Node binary | Framework | Notes |
| ------- | ----------- | --------- | ----- |
| [v11.5.1](./mainnet/v11.5.1) | `supra_node_v11.5.1` | `aptosvm-v1.16_supra-v1.8.15` | config, features, `supra` CLI, framework |
| [v11.4.3](./mainnet/v11.4.3) | `supra_node_v11.4.3` | `aptosvm-v1.16_supra-v1.8.13` | config, REST API, both CLIs, framework |
| [v11.3.6](./mainnet/v11.3.6) | `supra_node_v11.3.6` | `aptosvm-v1.16_supra-v1.8.9`  | config, REST API, both CLIs, framework |

## Reading these

Each release documents the changes **since the previous release on the same
network**, so upgrading across several versions means reading each one in turn,
oldest first. Start from a release's `README.md`: it pins the exact binary and
framework versions that make up that release and lists which surfaces changed.

Note that the framework changelogs chain on their own version series
(`aptosvm-v1.16_supra-*`) rather than on the node binary version, since the
framework is released independently.

## Related

- [Node configuration reference](../operations/node-configuration) — every field,
  with mainnet templates. It tracks the current release; check out this
  repository at an earlier tag to read it as it stood then.
- [Observability](../operations/observability) — monitoring your nodes.
