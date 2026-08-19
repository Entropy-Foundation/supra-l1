# Supra Node Configuration Guide

This document explains how to configure a Supra validator node. The node is typically started using:

```sh
target/release/supra node smr run
```

A Supra node is configured through **two** TOML files:

| File | Purpose |
|------|---------|
| `smr_settings.toml` | Per-node runtime settings: networking, database, transaction backlog, profiling, execution hooks. These settings are local to each node operator and may differ between nodes. |
| `genesis_parameters.toml` | Network-wide genesis parameters: chain identity, consensus, mempool, DKG, commitments, MoveVM economics, automation, and leader ban registry. These parameters must be agreed upon by all validators during the Genesis ceremony. |

Both files are expected to be located in the node's home directory (typically `$SUPRA_HOME`).

---

## Table of Contents
- [Supra Node Configuration Guide](#supra-node-configuration-guide)
  - [Table of Contents](#table-of-contents)
  - [smr\_settings.toml](#smr_settingstoml)
    - [\[node\]](#node)
      - [Sizing `dkg_thread_pool_size`](#sizing-dkg_thread_pool_size)
    - [\[node.database\_setup\]](#nodedatabase_setup)
    - [\[node.ws\_server\]](#nodews_server)
    - [\[node.backlog\_parameters\]](#nodebacklog_parameters)
    - [\[profiling\]](#profiling)
    - [\[executor\_hook\_config\]](#executor_hook_config)
    - [`prometheus_exporter_port`](#prometheus_exporter_port)
    - [Console ports](#console-ports)
    - [\[p2p\_auth\]](#p2p_auth)
    - [Complete smr\_settings.toml Example](#complete-smr_settingstoml-example)
  - [genesis\_parameters.toml](#genesis_parameterstoml)
    - [\[instance\]](#instance)
    - [\[moonshot\]](#moonshot)
    - [\[mempool\]](#mempool)
    - [\[dkg\]](#dkg)
    - [\[commitments\]](#commitments)
    - [\[move\_vm\]](#move_vm)
    - [\[automation\]](#automation)
      - [V2 Parameters](#v2-parameters)
    - [\[leader\_ban\_registry\]](#leader_ban_registry)
    - [Complete genesis\_parameters.toml Example](#complete-genesis_parameterstoml-example)

---

## smr_settings.toml

The main runtime settings file for the Supra validator node. It configures networking, database, transaction backlog, profiling, and execution hooks.

### [node]

Top-level node parameters.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `connection_refresh_timeout_sec` | integer | `20` | Interval (in seconds) at which the node verifies its connectivity to other nodes. |
| `rpc_access_port` | integer | `26000` | The listening port on the validator node for RPC requests. |
| `is_transaction_provider_trusted` | boolean | `false` | If `true`, the node will not validate transactions received directly. Validation still occurs during consensus voting and execution. Set to `false` unless you fully trust the transaction provider. |
| `dkg_thread_pool_size` | integer | *number of physical CPU cores* | Number of threads used for compute-heavy DKG operations, chiefly verifying the dealings broadcast by other validators at an epoch boundary. Must be at least `1`; the node refuses to start on `0`. Values above the host's logical core count are clamped to it, with a warning in the log. See the note below before changing it. |

#### Sizing `dkg_thread_pool_size`

This pool is not the node's only CPU consumer, and the three are sized independently — their
sum can exceed the host's core count:

| Pool | Sized by | Threads |
|---|---|---|
| DKG verification pool | `dkg_thread_pool_size` | physical cores, by default |
| Async runtime (Tokio) | not configurable | logical cores |
| Move execution pool | not configurable | logical cores |

The DKG pool is idle for most of an epoch and then saturates during the epoch transition:
every dealer broadcasts at once, so the whole batch of dealing verifications queues at
once and occupies every worker for seconds, competing with block execution on the same
cores. The larger the validator set, the more verifications land in that burst.

Guidance:

- **One validator per host, cores to spare:** leave the field out. The physical-core default
  finishes the burst fastest.
- **Host shared with an RPC node, or a small/hyperthreaded machine:** set it below the
  physical core count (for example half) to leave headroom for block execution during the
  epoch transition. The DKG takes longer but does not stall the executor.
- **Several validators on one host (development only):** divide the physical cores between
  them, since they all transition at the same time.

**Example:**
```toml
[node]
connection_refresh_timeout_sec = 10
rpc_access_port = 26000
is_transaction_provider_trusted = false
dkg_thread_pool_size = 4
```

### [node.database_setup]

Configures persistent storage for the node. The validator node requires at least a `chain_store` database instance.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allow_missing_data` | boolean | `false` | If `true`, allows the node to start even if expected data is missing from the database. |

**[node.database_setup.dbs.chain_store.rocks_db]** — Stores the blockchain data:

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | string | Filesystem path for the chain store RocksDB database. |
| `enable_pruning` | boolean | If `true`, old data is pruned based on `epochs_to_retain`. |
| `enable_snapshots` | boolean | If `true`, enables periodic database snapshots. |

**[node.database_setup.prune_config]** — Pruning configuration:

| Parameter | Type | Description |
|-----------|------|-------------|
| `epochs_to_retain` | integer | Number of epochs of data to retain. Must meet the system minimum. |

**Example:**
```toml
[node.database_setup]
allow_missing_data = false

[node.database_setup.dbs.chain_store.rocks_db]
path = "/node/smr_storage"
enable_pruning = true

[node.database_setup.prune_config]
epochs_to_retain = 2
```

### [node.ws_server]

Configures the WebSocket server used for RPC node synchronization.

**[node.ws_server.certificates]** — TLS certificates for the WebSocket server:

| Parameter | Type | Description |
|-----------|------|-------------|
| `root_ca_cert_path` | string | Path to the root CA certificate file. |
| `cert_path` | string | Path to the server TLS certificate file. |
| `private_key_path` | string | Path to the server private key file. |

**Example:**
```toml
[node.ws_server.certificates]
root_ca_cert_path = "/node/ca_certificate.pem"
cert_path = "/node/server_supra_certificate.pem"
private_key_path = "/node/server_supra_key.pem"
```

Two optional parameters sit directly under `[node.ws_server]`:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `transaction_forward_channel_capacity` | integer | `102400` | Capacity of the per-connection channel buffering outbound sync messages (certified blocks, transaction-inclusion certificates, committee authorizations) to a downstream RPC node. Messages are **dropped** when the buffer is full, so size it for the expected sync fan-out. |
| `termination_policy` | see below | `ignore_failed` | Whether to close a WebSocket connection after repeated failed transmissions. |

`termination_policy` takes one of two forms:

```toml
[node.ws_server]
termination_policy = "ignore_failed"        # never close on failed sends (default)
```

```toml
[node.ws_server.termination_policy]
terminate_after = 100                        # close after this many failed sends
```

### [node.backlog_parameters]

Controls the transaction backlog (the queue of pending transactions).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_backlog_size_mb` | integer | `16000` | Maximum allowed backlog size in megabytes. Based on total serialization size of queued elements. |
| `max_backlog_transactions_per_account` | integer | `25` | Maximum number of transactions a single account can have in the backlog at any time. |
| `max_backlog_transaction_time_to_live_seconds` | integer | `600` | Maximum allowed TTL (time-to-live) for transactions in the backlog, in seconds. Transactions exceeding this limit will not be scheduled, preventing DoS attacks with excessively long expiration timestamps. |

**Example:**
```toml
[node.backlog_parameters]
max_backlog_size_mb = 16000
max_backlog_transactions_per_account = 25
max_backlog_transaction_time_to_live_seconds = 600
```

### [profiling]

Optional profiling server for performance diagnostics. Only enable if you are debugging or profiling the node.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enabled` | boolean | `false` | Whether to enable the profiling server. |
| `host` | string | `"127.0.0.1"` | Host address to bind the profiling server to. |
| `port` | integer | `9876` | Port to bind the profiling server to. |

**Example:**
```toml
[profiling]
enabled = true
host = "127.0.0.1"
port = 9876
```

### [executor_hook_config]

Configures execution hooks that run at specific block heights for diagnostics or custom operations.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `block_height` | integer | `0` | Starting block height at which to execute the hook. |
| `interval` | integer | *none* (optional) | Interval in block heights between hook executions, starting from `block_height`. |
| `count` | integer | *none* (optional) | Total number of times the hook should run. |
| `output_dir` | string | *none* (optional) | Output directory to store hook results. |

**Example:**
```toml
[executor_hook_config]
block_height = 0
```

### `prometheus_exporter_port`

TCP port on which the validator binds its Prometheus metrics endpoint (`0.0.0.0:<port>`). Useful when a validator is co-located with an RPC node — both default to `9000`, so one must be changed to avoid a port collision.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `prometheus_exporter_port` | integer | `9000` | Prometheus metrics port. |

**Example:**
```toml
prometheus_exporter_port = 9001
```

### Console ports

Both are optional and bound on localhost. When omitted, an ephemeral port is chosen at startup;
set them explicitly to reach the consoles on known ports and to avoid a startup port-collision
race on hosts running many node processes.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `tokio_console_port` | integer | *ephemeral* | Port for the async-runtime `tokio-console` subscriber. |
| `tcp_console_port` | integer | *ephemeral* | Port for the node's `tcp_console` admin console — log-filter reload, network status, and on-demand database dump. |

**Example:**
```toml
tokio_console_port = 6669
tcp_console_port   = 6670
```

### [p2p_auth]

Optional. Configures external peer authentication against an authentication smart contract. Omit
the whole section unless your deployment uses one; a validator that omits it performs no external
peer authentication.

| Parameter | Type | Description |
|-----------|------|-------------|
| `auth_sc_address` | string | Address of the authentication smart contract. Must be `0x`-prefixed, valid hexadecimal, and of even length. |
| `auth_sc_client` | string | URL of the client used to query the contract. Must begin with `http://` or `https://`. |

Both are validated at startup, and the node refuses to start if either is malformed.

**Example:**
```toml
[p2p_auth]
auth_sc_address = "0x1234abcd..."
auth_sc_client  = "https://auth.example.com"
```

---

### Complete smr_settings.toml Example

```toml
[node]
connection_refresh_timeout_sec = 10
rpc_access_port = 26000
is_transaction_provider_trusted = false
dkg_thread_pool_size = 4

[node.database_setup]
allow_missing_data = false

[node.database_setup.dbs.chain_store.rocks_db]
path = "/node/smr_storage"
enable_pruning = true

[node.database_setup.prune_config]
epochs_to_retain = 2

[node.ws_server.certificates]
root_ca_cert_path = "/node/ca_certificate.pem"
cert_path = "/node/server_supra_certificate.pem"
private_key_path = "/node/server_supra_key.pem"

[node.backlog_parameters]
max_backlog_size_mb = 16000
max_backlog_transactions_per_account = 25
max_backlog_transaction_time_to_live_seconds = 600

[profiling]
enabled = false
host = "127.0.0.1"
port = 9876

[executor_hook_config]
block_height = 0

prometheus_exporter_port = 9000
```

---

## genesis_parameters.toml

The consolidated Genesis parameters file. It contains all network-wide parameters that must be agreed upon by all validators during the Genesis ceremony. After Genesis, most of these parameters can only be updated via governance actions.

This file groups the following logical sections under a single TOML file:

| Section | Subsystem |
|---------|-----------|
| `[instance]` | Chain identity, epoch, and genesis timestamp |
| `[moonshot]` | Moonshot consensus protocol parameters |
| `[mempool]` | Transaction batching and mempool synchronization |
| `[dkg]` | Distributed Key Generation parameters |
| `[commitments]` | Commitments protocol retry configuration |
| `[move_vm]` | MoveVM staking, governance, and economics |
| `[automation]` | Supra native automation (task scheduling) |
| `[leader_ban_registry]` | Leader ban registry (proposer ban policy) |

---

### [instance]

Defines the chain identity and epoch structure.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `chain_id` | integer | `255` (localnet) | Unique identifier for this blockchain instance. Prevents replay attacks across different chain deployments. |
| `epoch_duration_secs` | integer | `7200` | Minimum number of seconds in a consensus epoch. A new epoch begins when the first block with a timestamp exceeding the epoch start + this duration is executed. |
| `recurring_lockup_duration_secs` | integer | `14400` | Duration (in seconds) for automatic stake lockup renewal. Must be greater than 0 and at least as long as `epoch_duration_secs`. |
| `voting_duration_secs` | integer | `7200` | Voting period for governance proposals (in seconds). Must be strictly smaller than `recurring_lockup_duration_secs`. |
| `is_testnet` | boolean | `false` | Controls whether test features (e.g., faucet) are enabled at genesis. Should be `false` for mainnet deployments. |
| `genesis_timestamp_microseconds` | integer | `1728518400000000` | Unix timestamp in microseconds that denotes the genesis of the network. Fed to the MoveVM as a genesis configuration. |

**Example:**
```toml
[instance]
chain_id = 8
epoch_duration_secs = 7200
recurring_lockup_duration_secs = 14400
voting_duration_secs = 7200
is_testnet = false
genesis_timestamp_microseconds = 1732060800000000
```

---

### [moonshot]

Configures the Moonshot consensus protocol, which governs block production, leader election, timeouts, and synchronization.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `block_recency_bound_ms` | integer | `500` | Maximum number of milliseconds that the timestamp of a proposed block may be ahead of the local node's time. Prevents Byzantine leaders from forcing honest nodes to wait indefinitely by proposing blocks with timestamps far in the future. |
| `halt_block_production_when_no_txs` | boolean | `false` | If `true`, the proposer stops producing blocks when there are no transactions to process, conserving disk space. All nodes should use the same value. |
| `leader_elector` | string | `"FairSuccession"` | Leader election algorithm. Options: `"FairSuccession"`, `"RoundRobin"`. |
| `max_block_delay_ms` | integer | `1250` | Delay (in milliseconds) after which the block proposer will create a new block even if there are no payload items to propose. |
| `max_payload_items_per_block` | integer | `50` | Maximum number of payload items (mempool certificates) that may be included in a single consensus block. |
| `message_recency_bound_rounds` | integer | `20` | Number of rounds ahead of the current round for which the node will accept proposals, votes, and timeouts. Must be the same for all nodes. |
| `sync_retry_delay_ms` | integer | `1000` | Delay (in milliseconds) before retrying a sync request. Should be the same for all nodes. |
| `timeout_delay_ms` | integer | `3500` | Time (in milliseconds) after which the consensus core sends a Timeout message for the current round, measured from the start of the round. Must be the same for all nodes. |

**Example:**
```toml
[moonshot]
block_recency_bound_ms = 500
halt_block_production_when_no_txs = false
leader_elector = "FairSuccession"
max_block_delay_ms = 1250
max_payload_items_per_block = 50
message_recency_bound_rounds = 20
sync_retry_delay_ms = 1000
timeout_delay_ms = 3500
```

---

### [mempool]

Configures the mempool, which manages transaction batching and synchronization before transactions are proposed for consensus.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `min_batch_delay_ms` | integer | *none* (optional) | Minimum delay (in milliseconds) between two consecutive batches, even if the next batch is ready. If omitted, no minimum delay is enforced. |
| `max_batch_delay_ms` | integer | `500` | Maximum delay (in milliseconds) after which a batch is sealed, even if `max_batch_size_bytes` has not been reached. |
| `max_batch_size_bytes` | integer | `5000000` | Maximum size of a transaction batch in bytes (5 MB). Sized to support Supra Framework updates. |
| `max_payload_items_per_batch` | integer | `200` | Maximum number of transactions that may be included in a single batch. |
| `sync_retry_delay_ms` | integer | `2000` | Delay (in milliseconds) before retrying sync requests. Should be the same for all nodes. |
| `sync_retry_nodes` | integer | `3` | Number of randomly selected committee nodes to sync with when retrying sync requests. |

**Example:**
```toml
[mempool]
max_batch_delay_ms = 500
max_batch_size_bytes = 5000000
max_payload_items_per_batch = 200
sync_retry_delay_ms = 2000
sync_retry_nodes = 3
```

---

### [dkg]

Configures the Distributed Key Generation (DKG) protocol, which is responsible for generating shared cryptographic keys across validators at epoch boundaries.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `dealing_signature_collection_timeout_ms` | integer | `3000` | Time (in milliseconds) that a dealer waits to receive extra signatures on their dealing before proceeding. |
| `data_retention_epochs` | integer | `3` | Number of epochs to retain DKG data in the database for node syncing purposes. Consensus can currently only start syncing from the start of the previous epoch. |
| `dkg_timeout_ms` | integer | `300000` | Overall DKG process timeout (in milliseconds). If this timeout is hit, the DKG process will restart to try and make progress. |

**Example:**
```toml
[dkg]
dealing_signature_collection_timeout_ms = 3000
data_retention_epochs = 3
dkg_timeout_ms = 300000
```

---

### [commitments]

Configures the commitments protocol, which handles transaction inclusion certification and certificate synchronization.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `proposal_retry_delay_ms` | integer | `500` | Retry delay (in milliseconds) for the Transactions Inclusion Certifier. Also used by the Certificates Synchronization component as the proposal certificates request retry delay. |

**Example:**
```toml
[commitments]
proposal_retry_delay_ms = 500
```

---

### [move_vm]

Configures the MoveVM staking, governance, and economics parameters used during genesis transaction generation. These parameters establish the initial network economics and can be updated later via governance.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `min_stake` | integer | `0` | Minimum stake (in Quants, where 1 SUPRA = 10^8 Quants) required to run a validator. Set to 0 at genesis because The Foundation's stake is added after pool creation; intended to be increased to 55M SUPRA via governance. A governance change takes effect at the start of the next epoch, not immediately, and the minimum is compared against a validator's staked balance alone — pending rewards do not count towards it. |
| `max_stake` | string | `"10000000000000000000"` | Maximum stake a validator can possess. Serialized as a string due to a limitation in the TOML crate with values larger than `i64::MAX`. |
| `validator_commission_rate_percentage` | integer | `3774` | Commission rate for validators, specified as a percentage with 2 decimals of precision (e.g., `3774` = 37.74%). |
| `voting_power_increase_limit` | integer | `33` | Maximum voting power increase limit for governance after every epoch. |
| `rewards_apy_percentage` | integer | `1285` | Staking rewards APY, specified as a percentage with 2 decimals of precision (e.g., `1285` = 12.85%). |
| `remaining_balance_lockup_cliff_period_in_seconds` | integer | `604800` | Cliff period (in seconds) for initial balance lockup vesting pools. Default is 7 days. |
| `operator_account_balance` | integer | `0` | Account balance (in Quants) to be minted for operator addresses. |
| `required_proposer_stake` | integer | `0` | Minimum stake required to create governance proposals. |
| `allow_new_validators` | boolean | `false` | Whether new validators are allowed to join the set after genesis. |
| `pbo_owner_stake` | integer | `5500000000000000` | Amount of SUPRA Quants that the PBO (Pool-Based Owner) account will stake after the PBO pools are created. Default is 55M SUPRA. |

**Validation Rules:**
- `min_stake` must not be greater than `max_stake`.
- `max_stake` cannot exceed the total supply (100 billion SUPRA).
- `validator_commission_rate_percentage` cannot exceed `10000` (100.00%).
- `rewards_apy_percentage` cannot exceed `10000` (100.00%).

**Example:**
```toml
[move_vm]
min_stake = 0
max_stake = "10000000000000000000"
validator_commission_rate_percentage = 3774
voting_power_increase_limit = 33
rewards_apy_percentage = 1285
remaining_balance_lockup_cliff_period_in_seconds = 604800
operator_account_balance = 0
required_proposer_stake = 0
allow_new_validators = false
pbo_owner_stake = 5500000000000000
```

---

### [automation]

Configures the Supra native automation feature, which provides on-chain task scheduling capabilities. These parameters are used during genesis to set up the automation registry. Future updates should be done via governance.

The configuration uses a versioned envelope — either `V1` or `V2`. V2 is recommended for new deployments as it adds support for system tasks and cycle-based scheduling.

| Parameter | Type | Description |
|-----------|------|-------------|
| `enable_native_automation_feature` | boolean | If `true`, the native automation feature flag is enabled at genesis. If `false`, the feature can be enabled later via governance. |

#### V2 Parameters

The `[automation.V2.parameters]` table configures the automation registry:

| Parameter | Type | Description |
|-----------|------|-------------|
| `task_duration_cap_in_secs` | integer | Maximum duration (in seconds) for an automation task. Must be greater than `cycle_duration_secs`. |
| `registry_max_gas_cap` | integer | Maximum gas cap for the automation registry. Must be greater than 0. |
| `automation_base_fee_in_quants_per_sec` | integer | Base fee charged for automation tasks, denominated in Quants per second. |
| `flat_registration_fee_in_quants` | integer | One-time flat fee charged when registering an automation task. |
| `congestion_threshold_percentage` | integer | Percentage threshold (0–100) of task capacity utilization at which congestion fees begin to apply. |
| `congestion_base_fee_in_quants_per_sec` | integer | Additional fee per second applied when the congestion threshold is exceeded. |
| `congestion_exponent` | integer | Exponent used in the congestion fee calculation. Must be greater than 0. Higher values result in steeper fee increases. |
| `task_capacity` | integer | Maximum number of user automation tasks the registry can hold. |
| `cycle_duration_secs` | integer | Duration (in seconds) of a single automation cycle. Must be less than `task_duration_cap_in_secs`. |
| `system_task_duration_cap_in_secs` | integer | Maximum duration (in seconds) for a system automation task. Must be greater than `cycle_duration_secs`. |
| `system_tasks_max_gas_cap` | integer | Maximum gas cap for system tasks. Must be greater than 0. |
| `system_task_capacity` | integer | Maximum number of system automation tasks the registry can hold. |

**Validation Rules:**
- If `enable_native_automation_feature` is `true`, `parameters` must be provided.
- `congestion_threshold_percentage` must be in the range `[0, 100]`.
- `congestion_exponent` must be greater than 0.
- `registry_max_gas_cap` must be greater than 0.
- `task_duration_cap_in_secs` must be greater than `cycle_duration_secs`.
- `system_task_duration_cap_in_secs` must be greater than `cycle_duration_secs`.
- `system_tasks_max_gas_cap` must be greater than 0.

**Example (V2):**
```toml
[automation.V2]
enable_native_automation_feature = true

[automation.V2.parameters]
task_duration_cap_in_secs = 7200
registry_max_gas_cap = 100000
automation_base_fee_in_quants_per_sec = 4000000
flat_registration_fee_in_quants = 50000000
congestion_threshold_percentage = 50
congestion_base_fee_in_quants_per_sec = 4000000
congestion_exponent = 6
task_capacity = 500
cycle_duration_secs = 30
system_task_duration_cap_in_secs = 7200
system_tasks_max_gas_cap = 50000
system_task_capacity = 100
```

**Example (V1 — legacy):**
```toml
[automation.V1]
enable_native_automation_feature = true

[automation.V1.parameters]
task_duration_cap_in_secs = 2626560
registry_max_gas_cap = 100000000
automation_base_fee_in_quants_per_sec = 1000
flat_registration_fee_in_quants = 100000000
congestion_threshold_percentage = 80
congestion_base_fee_in_quants_per_sec = 100
congestion_exponent = 6
task_capacity = 500
```

---

### [leader_ban_registry]

Configures the leader ban registry, which temporarily bans validators from being elected as consensus leaders when they fail to produce a committed block during their turn. Escalating bans apply for consecutive failures, followed by a probation period.

The configuration uses a versioned envelope. Currently only `V0` is supported. All parameters are expressed in units of *elections*, where **one election equals one full rotation through the committee** (i.e., `1 election = committee_size` consensus rounds).

| Parameter | Type | Description |
|-----------|------|-------------|
| `initial_elections_denied` | integer | Number of elections for which a validator is banned from being elected as consensus leader after their first failure to produce a committed block. Ban duration in rounds = `committee_size * initial_elections_denied`. Set to `0` to disable the ban registry entirely. |
| `max_elections_denied` | integer | Maximum number of elections a validator can be banned after consecutive failures. Ban durations escalate from `initial_elections_denied` up to this cap. |
| `minimum_unbanned_proposers` | integer | Minimum number of validators that must remain unbanned at any time. If applying a new ban would reduce the eligible proposer count below this threshold, the ban is not applied. This safeguards liveness when many validators misbehave simultaneously. |
| `probation_elections` | integer | Number of elections a validator spends on probation after their ban expires. During probation the validator may propose, but a subsequent failure escalates their ban. |

**Example:**
```toml
[leader_ban_registry.V0]
initial_elections_denied = 1
max_elections_denied = 50
minimum_unbanned_proposers = 1
probation_elections = 5
```

---

### Complete genesis_parameters.toml Example

```toml
[instance]
chain_id = 255
epoch_duration_secs = 60
recurring_lockup_duration_secs = 14400
voting_duration_secs = 7200
is_testnet = true
genesis_timestamp_microseconds = 0

[moonshot]
block_recency_bound_ms = 500
halt_block_production_when_no_txs = false
leader_elector = "FairSuccession"
max_block_delay_ms = 1250
max_payload_items_per_block = 50
message_recency_bound_rounds = 20
sync_retry_delay_ms = 1000
timeout_delay_ms = 3500

[mempool]
max_batch_delay_ms = 500
max_batch_size_bytes = 5000000
max_payload_items_per_batch = 200
sync_retry_delay_ms = 2000
sync_retry_nodes = 3

[dkg]
dealing_signature_collection_timeout_ms = 3000
data_retention_epochs = 3
dkg_timeout_ms = 300000

[commitments]
proposal_retry_delay_ms = 500

[move_vm]
min_stake = 0
max_stake = "10000000000000000000"
validator_commission_rate_percentage = 3774
voting_power_increase_limit = 33
rewards_apy_percentage = 1285
remaining_balance_lockup_cliff_period_in_seconds = 604800
operator_account_balance = 0
required_proposer_stake = 0
allow_new_validators = false
pbo_owner_stake = 5500000000000000

[automation.V2]
enable_native_automation_feature = true

[automation.V2.parameters]
task_duration_cap_in_secs = 7200
registry_max_gas_cap = 100000
automation_base_fee_in_quants_per_sec = 4000000
flat_registration_fee_in_quants = 50000000
congestion_threshold_percentage = 50
congestion_base_fee_in_quants_per_sec = 4000000
congestion_exponent = 6
task_capacity = 500
cycle_duration_secs = 30
system_task_duration_cap_in_secs = 7200
system_tasks_max_gas_cap = 50000
system_task_capacity = 100

[leader_ban_registry.V0]
initial_elections_denied = 1
max_elections_denied = 50
minimum_unbanned_proposers = 1
probation_elections = 5
```
