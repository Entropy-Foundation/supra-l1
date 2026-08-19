# Mainnet v11.5.1

| | |
| --- | --- |
| **Node binary** | `supra_node_v11.5.1` |
| **Move framework / VM** | `aptosvm-v1.16_supra-v1.8.15` |
| **Previous mainnet release** | [v11.4.3](../v11.4.3) |

Changes below are relative to that previous release. Upgrading from further back
means reading each intervening release in turn.

| Surface | Changes since |
| ------- | ------------- |
| [Node configuration](./config.md) | `supra_node_v11.4.3` |
| [`supra` CLI](./supra_cli.md) | `supra_node_v11.4.3` |
| [Supra framework](./supra_framework.md) | `aptosvm-v1.16_supra-v1.8.13` |
| [Feature activation](./features.md) | *governance step — see below* |

The RPC node CLI and the REST API are unchanged in this release.

> **Feature activation is a separate step.** The binary ships the code; a
> governance proposal turns the feature flags on afterwards. Nothing in
> [features.md](./features.md) takes effect until that happens.

The configuration reference for this release is the
[node configuration guide](../../../operations/node-configuration) as of this
repository's `v11.5.1` tag.
