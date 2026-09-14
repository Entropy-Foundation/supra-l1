# Mainnet v11.7.0

| | |
| --- | --- |
| **Node binary** | `supra_node_v11.7.0` |
| **Move framework / VM** | `aptosvm-v1.16_supra-v1.8.17` |
| **Previous mainnet release** | [v11.5.1](../v11.5.1) |

Changes below are relative to that previous release. Upgrading from further back
means reading each intervening release in turn.

| Surface | Changes since |
| ------- | ------------- |
| [Node configuration](./config.md) | `supra_node_v11.5.1` |
| [REST API](./rpc_api.md) | `supra_node_v11.5.1` |
| [On-disk storage](./storage.md) | `supra_node_v11.5.1` |
| [`supra` CLI](./supra_cli.md) | `supra_node_v11.5.1` |

The RPC node CLI is unchanged in this release, and no feature flags are added
or activated by it.

> **This release changes the on-disk format.** A database written by this
> release cannot be read by an earlier one, so the upgrade is one way per
> database and a snapshot taken after it is of no use to a node still on an
> earlier release. Read [storage.md](./storage.md) before upgrading, and before
> restoring any node from a snapshot.

Two configuration fields become **required** in this release — the validator's
`node.ws_server.certificates.root_ca_cert_path` and the RPC node's
`synchronization.ws.certificates.root_ca_cert_path`. A node that omits either no
longer starts. See [config.md](./config.md).

The configuration reference for this release is the
[node configuration guide](../../../operations/node-configuration) as of this
repository's `v11.7.0` tag.
