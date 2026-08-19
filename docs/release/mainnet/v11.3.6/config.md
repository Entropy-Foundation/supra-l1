# Node Configuration Change Log

Changes to node configuration files since `supra_node_v10.0.8`.

---

## Configuration File Locations

**Breaking change.** All configuration files are now resolved relative to the `SUPRA_HOME` directory. Node operators that previously relied on a hardcoded path for `config.toml` or `smr_settings.toml` must update their tooling to resolve files against this directory.

---

## `genesis_parameters.toml` — New File

A new top-level configuration file, `genesis_parameters.toml`, has been introduced. It holds the network parameters that all validators must agree on before the network starts (genesis ceremony parameters). Previously these parameters were embedded inside `smr_settings.toml`; they now live in their own file.

### Fields migrated from `smr_settings.toml`

The following sections were removed from `smr_settings.toml` and must now be placed in `genesis_parameters.toml`:

| Section | Description |
|---|---|
| `instance` | Chain identity parameters: chain ID, epoch duration, recurring lock-up duration, voting duration, and genesis timestamp. |
| `mempool` | Mempool tuning parameters. |
| `moonshot` | Consensus protocol parameters. |
| `move_vm` | Move VM execution parameters including stake bounds, validator commission rates, and rewards configuration. |
| `automation` | Parameters for the on-chain automation engine. |

### New sections in `genesis_parameters.toml`

The following sections are new and have no equivalent in previous releases:

#### `[commitments]`

Parameters for the transaction-inclusion proof protocol.

| Field | Type | Default | Description |
|---|---|---|---|
| `proposal_retry_delay_ms` | integer (ms) | `500` | How long to wait before retrying a proof proposal or certificate synchronisation request. |

#### `[dkg]`

Parameters for Distributed Key Generation (DKG), the protocol used to produce the network's threshold signing key at the start of each epoch.

| Field | Type | Default | Description |
|---|---|---|---|
| `dealing_signature_collection_timeout_ms` | integer (ms) | `3000` | How long a dealer waits to collect additional signatures on its dealing. |
| `data_retention_epochs` | integer | `3` | Number of past epochs for which DKG data is retained to support node synchronisation. |
| `dkg_timeout_ms` | integer (ms) | `100000` | Overall DKG completion timeout. If this is exceeded the DKG process restarts automatically. |

#### `[leader_ban_registry]`

Parameters governing how the consensus protocol temporarily bans validators that fail to propose committed blocks when elected as leader.

| Field | Type | Description |
|---|---|---|
| `initial_elections_denied` | integer (u8) | Initial number of election opportunities denied to a validator after a proposal failure. Scaled by committee size to keep the ban duration roughly constant across different validator set sizes. |
| `max_elections_denied` | integer (u32) | Maximum number of election opportunities that can be denied for consecutive proposal failures. |
| `minimum_unbanned_proposers` | integer (u8) | Lower bound on the number of validators that must remain eligible for election. A ban is not applied if it would drop the eligible count below this value. |
| `probation_elections` | integer (u8) | Number of elections a validator must complete on probation after its ban expires. Failing during probation increments the consecutive-ban counter; successfully completing probation clears the ban record. |

---

## `smr_settings.toml` — Validator Node Settings

### Removed fields

The sections `instance`, `mempool`, `moonshot`, `move_vm`, and `automation` have been moved to `genesis_parameters.toml` (see above). They are no longer read from `smr_settings.toml`.

### New section: `[executor_hook_config]`

An optional section that configures post-block-execution hooks. When set, the node performs a configurable action (such as dumping selected database contents) after executing the specified block height(s). All fields default to zero/absent, which disables the hook.

| Field | Type | Default | Description |
|---|---|---|---|
| `block_height` | integer | `0` | Block height at which to first trigger the hook. |
| `interval` | integer \| null | absent | If set, the hook repeats every `interval` blocks after the start height. |
| `count` | integer \| null | absent | If set, limits the total number of times the hook runs. |
| `output_dir` | string \| null | absent | Directory to write output to. |

### New field: `prometheus_exporter_port`

| Field | Type | Default | Description |
|---|---|---|---|
| `prometheus_exporter_port` | integer (port) | `9000` | TCP port on which the validator binds its Prometheus metrics endpoint (`0.0.0.0:<port>`). Can be set to a non-default value when a validator is co-located with an RPC node to avoid a collision on the shared default port. |

---

## `config.toml` — RPC Node Settings

### New section: `[executor_hook_config]`

Identical in structure and purpose to the same section in `smr_settings.toml` (see above). Allows RPC nodes to trigger post-execution hooks at specified block heights.

### New field: `max_certificates_in_backlog`

| Field | Type | Default | Description |
|---|---|---|---|
| `max_certificates_in_backlog` | integer | `1000` | Maximum number of transaction certificates the backlog may hold ahead of the last verified block height. For example, with a value of `1000` and a latest stored certificate height of `200`, certificates above height `1200` are rejected. |

### New field: `[faucet].receiver_balance_uppercap`

| Field | Type | Description |
|---|---|---|
| `receiver_balance_uppercap` | integer (quants) | Maximum quant balance a receiver may hold to be eligible for faucet funding. Requests are rejected at the API layer if the receiver's on-chain balance meets or exceeds this value. |

### New fields: `[websocket_limits]`

Two new optional fields controlling the Supra-native WebSocket endpoint (`/rpc/v4/ws`):

| Field | Type | Default | Description |
|---|---|---|---|
| `enable_supra_websocket` | boolean | `true` | Whether the Supra-native WebSocket endpoint (`/rpc/v4/ws`) accepts connections. Toggling to `false` disables the endpoint without affecting the EVM WebSocket endpoint (`/rpc/v1/eth`). |
| `max_outbound_queue` | integer | `256` | Bounded capacity (in messages) of each connection's outbound notification queue. When the queue is full the newest notifications are dropped rather than allowing the queue to grow without bound, protecting against slow consumers. |
