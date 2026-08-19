# Supra RPC Node Configuration Guide

This document explains how to configure a Supra RPC node using the `config.toml` file. This guide describes each parameter, its options, and how to use them.

---

## Table of Contents
- [Supra RPC Node Configuration Guide](#supra-rpc-node-configuration-guide)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Parameter Reference](#parameter-reference)
    - [Protocol Parameters (`chain_instance`)](#protocol-parameters-chain_instance)
    - [Node Parameters](#node-parameters)
    - [Synchronization](#synchronization)
      - [WebSocket Synchronization (`ws`)](#websocket-synchronization-ws)
      - [REST Synchronization (`rest`)](#rest-synchronization-rest)
    - [Database Setup](#database-setup)
      - [\[database\_setup.dbs.archive.rocks\_db\]](#database_setupdbsarchiverocks_db)
      - [\[database\_setup.dbs.chain\_store.rocks\_db\]](#database_setupdbschain_storerocks_db)
      - [\[database\_setup.prune\_config\]](#database_setupprune_config)
    - [HTTP Server](#http-server)
    - [Faucet Settings](#faucet-settings)
    - [Limiting Strategy](#limiting-strategy)
    - [CORS / Allowed Origins](#cors--allowed-origins)
    - [Consensus Access Tokens](#consensus-access-tokens)
    - [Backlog Parameters](#backlog-parameters)
    - [Executor Hook](#executor-hook)
    - [Other Parameters](#other-parameters)
  - [Example](#example)

---

## Overview

The `config.toml` file configures all aspects of a Supra RPC node, including protocol, networking, database, security, and operational parameters.

The file is expected to be located in the the same directory as the `rpc_node` binary. However, if you are using the [manage_supra_nodes.sh](https://github.com/Entropy-Foundation/supra-nodeops-data/blob/master/node_management/manage_supra_nodes.sh) helper scripts (refer to the related docs page for [installation instructions](https://docs.supra.com/network/node/node-upgrade-guide-mainnet/download-the-node-management-scripts)) then `config.toml` is expected to be located in `<host_supra_home>`. The management script will automatically copy it into the Docker container when the node is started.

---

## Parameter Reference

### Protocol Parameters (`chain_instance`)
- **chain_id**: Unique identifier for the protocol instance. Prevents replay attacks. *(integer)*
- **epoch_duration_secs**: Epoch length in seconds. *(integer)*
- **recurring_lockup_duration_secs**: Duration (in seconds) for automatic stake lockup renewal. *(integer)*
- **voting_duration_secs**: Voting period for governance proposals (in seconds). *(integer)*
- **is_testnet**: Set to `true` for testnet, `false` for mainnet. *(boolean)*
- **genesis_timestamp_microseconds**: Genesis timestamp in microseconds since epoch. *(integer)*

### Node Parameters
- **bind_addr**: Address and port for incoming RPC requests (e.g., `0.0.0.0:30000`). *(string)*
- **block_provider_is_trusted**: If `true`, disables block verification before execution. Should be `false` unless you control the block provider. *(boolean)*
- **supra_committees_config**: Path to the `supra_committees.json` file. *(string, optional)*

### Synchronization

**synchronization**: Configures how the node synchronizes with the network. There are two main modes:

#### WebSocket Synchronization (`ws`)
Used for direct, real-time synchronization with a validator node over WebSocket. Validators require unique certificates for each client that they serve and generally act as their own Certificate Authorities. Accordingly, you must ask the operator of the validator that you wish to sync from to issue a certificate for your RPC node.

**Parameters:**
- `consensus_rpc` (string): WebSocket address of the validator (e.g., `ws://<VALIDATOR_IP>:26000`).
- `certificates` (table, optional): TLS configuration for secure connections.
  - `cert_path` (string): Path to the TLS certificate file.
  - `private_key_path` (string): Path to the private key file.
  - `root_ca_cert_path` (string): Path to the root CA certificate file.

**Example:**
```toml
[synchronization.ws]
consensus_rpc = "ws://127.0.0.1:26000"

[synchronization.ws.certificates]
cert_path = "./configs/client_certificate.pem"
private_key_path = "./configs/client_key.pem"
root_ca_cert_path = "./configs/ca_certificate.pem"
```

#### REST Synchronization (`rest`)
Used for polling another RPC node for new blocks via the HTTP REST API.

The height of each synced block is compared against the latest locally known block height. If the local node is behind, all missing intermediate blocks are requested and synchronized. Given that the Supra chain produces blocks at a high rate, the local node will, in practice, almost always lag by a few blocks. In this context, enabling `request_latest_height` will typically trigger a sync operation, causing two network requests to be issued: one to fetch the latest height and another to fetch the corresponding block(s). This behavior should be taken into account when setting this flag to `true`.

**Parameters:**
- `access_token` (string): Token used to authenticate requests to the REST API.
- `endpoint` (string): URL of the REST API endpoint (e.g., `http://<EXISTING_RPC_IP>:30000`).
- `poll_interval_milliseconds` (integer): How often (in ms) to poll for new blocks.
- `request_latest_height` (boolean): If `true`, requests the latest block height before polling for a new block (reduces data transfer when block production is slow). Generally recommended to be set to `false`.

**Example:**
```toml
[synchronization.rest]
access_token = "ExampleAccessTokenForRPCNode"
endpoint = "http://127.0.0.1:30000"
poll_interval_milliseconds = 300
request_latest_height = false
```

### Database Setup

**database_setup**: Configures persistent storage for the node. Each database uses RocksDB and can be tuned for pruning and snapshots.

An RPC node has exactly two storage instances, `chain_store` and `archive`, and both must be
configured. No other instance name is accepted: an unrecognised key under `database_setup.dbs`
fails to parse and the node will not start.

#### [database_setup.dbs.archive.rocks_db]
Stores indexes used to serve RPC API calls.
- `path` (string): Filesystem path for the archive DB (e.g., `./configs/rpc_archive`).
- `enable_pruning` (bool): If `true`, old data is pruned based on `epochs_to_retain`.
- `enable_snapshots` (bool): If `true`, enables periodic database snapshots.

#### [database_setup.dbs.chain_store.rocks_db]
Stores the blockchain data.
- `path` (string): Filesystem path for the chain store DB (e.g., `./configs/rpc_store`).
- `enable_pruning` (bool): If `true`, old data is pruned based on `epochs_to_retain`.
- `enable_snapshots` (bool): If `true`, enables periodic database snapshots.

#### [database_setup.prune_config]
Controls pruning for all databases where pruning is enabled.
- `epochs_to_retain` (integer): Number of epochs of data to retain. For example, `1008` epochs is about 3 months if an epoch is 2 hours.

**Example:**
```toml
[database_setup.dbs.archive.rocks_db]
path = "./configs/rpc_archive"
enable_pruning = true
enable_snapshots = false

[database_setup.dbs.chain_store.rocks_db]
path = "./configs/rpc_store"
enable_pruning = true
enable_snapshots = false

[database_setup.prune_config]
epochs_to_retain = 1008
```

### HTTP Server
**http_server**: (Optional) Advanced configuration for the built-in HTTP server. Most users can use the defaults. The main options are:

- `workers` (integer): Number of worker threads to start. Defaults to the number of CPU cores.
- `backlog` (integer): Maximum number of pending connections (clients waiting to be served). Default: 2048.
- `maxconn` (integer): Maximum per-worker number of concurrent connections. Default: 25,600.
- `maxconnrate` (integer): Maximum per-worker concurrent SSL connection establishment. Default: 256.
- `keep_alive` (integer): Server keep-alive setting in seconds. Default: 5.
- `disconnect_timeout` (integer): Timeout in seconds for disconnecting idle connections. Default: 1.
- `ssl_handshake_timeout` (integer): Timeout in seconds for SSL handshake. Default: 5.
- `headers_read_rate` (table): Controls request header read rate and timeouts.
  - `rate` (integer): Data rate threshold in bytes. Default: 256.
  - `timeout` (integer): Timeout in seconds for reading headers. Default: 5.
  - `max_timeout` (integer): Maximum timeout in seconds. Default: 15.
- `websocket_limits` (table): Limits for websocket connections and subscriptions.
  - `max_total_connections` (integer): Maximum total concurrent websocket connections. Default: 1000.
  - `max_connections_per_token` (integer): Maximum concurrent websocket connections per token (or anonymous). Default: 100.
  - `max_subscriptions_per_connection` (integer): Maximum active subscriptions per connection. Default: 100.
  - `websocket_auth_tokens` (array): List of allowed bearer tokens for websocket connections.
  - `enable_supra_websocket` (boolean): Whether the Supra-native WebSocket endpoint (`/rpc/v4/ws`) accepts connections. Independent of the EVM WebSocket endpoint (`/rpc/v1/eth`). Default: `true`.
  - `max_outbound_queue` (integer): Bounded capacity (in messages) of each connection's outbound notification queue. When full, the newest notifications are dropped to bound memory use for slow consumers. Default: `256`.

**Example:**
```toml
[http_server]
workers = 8
backlog = 2048
maxconn = 25600
maxconnrate = 256
keep_alive = 5
disconnect_timeout = 1
ssl_handshake_timeout = 5

[http_server.headers_read_rate]
rate = 256
timeout = 5
max_timeout = 15

[http_server.websocket_limits]
max_total_connections = 1000
max_connections_per_token = 100
max_subscriptions_per_connection = 100
websocket_auth_tokens = []
enable_supra_websocket = true
max_outbound_queue = 256
```

### Faucet Settings
**faucet_settings**: (Optional) Enables faucet mode, allowing the node to distribute testnet tokens to users for development and testing. Only relevant for testnet or devnet nodes. The main options are:

- `minters` (table): Source of minter accounts. Options:
  - `minter_count` (integer): Number of minter accounts to generate (for local/devnet only).
  - `minter_profiles_path` (string): Path to a file containing CLI profiles with mint capability (recommended for production testnets).
- `quants_granted_per_request` (integer): Number of tokens granted per faucet request. Must be > 0 and less than `max_daily_quants_per_user`.
- `max_daily_quants_per_user` (integer): Maximum number of tokens a user can receive per day.
- `cooldown_period_in_seconds` (integer): Cooldown period (in seconds) between requests from the same user.
- `receiver_balance_uppercap` (integer): Maximum quant balance a receiver may hold to be eligible for funding. Requests are rejected if the receiver's on-chain balance meets or exceeds this value.

**Example (local/devnet):**
```toml
[faucet_settings]
minters.minter_count = 10
quants_granted_per_request = 500000000
max_daily_quants_per_user = 1000000000
cooldown_period_in_seconds = 3600
receiver_balance_uppercap = 5000000000
```

**Example (production testnet):**
```toml
[faucet_settings]
minters.minter_profiles_path = "./profiles/minters.json"
quants_granted_per_request = 500000000
max_daily_quants_per_user = 1000000000
cooldown_period_in_seconds = 3600
receiver_balance_uppercap = 5000000000
```

### Limiting Strategy
**limiting_strategy**: (Optional) Controls rate-limiting for transaction submissions and/or faucet requests to prevent abuse. Two main strategies are supported:

- **Interval**: Allows one request per fixed interval per user/account.
  - `secs` (integer): Minimum interval (in seconds) between allowed requests from the same user/account.
  - `nanos` (integer): Additional interval (in nano seconds) between allowed requests from the same user/account. Added to `secs` to produce the final interval.

- **ExponentialBackoff**: Increases the wait time between allowed requests when requests are sent faster than the accepted rate.
  - `secs` (integer): Minimum interval (in seconds) between allowed requests from the same user/account.
  - `nanos` (integer): Additional interval (in nano seconds) between allowed requests from the same user/account. Added to `secs` to produce the final interval.
  - `factor` (float): Factor by which the delay increases after each request that fails to respect the specified interval (e.g., 2.0 doubles the delay each time).

**Example (Interval strategy):**
```toml
[limiting_strategy]
Interval.secs = 1  # Allow one request per second per user/account
Interval.nanos = 0
```

**Example (ExponentialBackoff strategy):**
```toml
[limiting_strategy]
ExponentialBackoff.secs = 5
ExponentialBackoff.nanos = 0
ExponentialBackoff.factor = 0.8
```

### CORS / Allowed Origins
- **allowed_origin**: List of allowed origins for CORS. Each entry includes:
  - **url**: Allowed origin URL. *(string)*
  - **description**: Description of the origin. *(string, optional)*
  - **mode**: Mode for the origin (`Server` or `Cors`). *(string, optional)*

### Consensus Access Tokens

**consensus_access_tokens**: List of access tokens for authenticating public RPC requests. Each token entry includes:
- **token_hash**: Keccak256 hash of the access token string. *(string)*
- **authorized_source**: IP address or hostname authorized to use this token. *(object: `ipaddr` or `host`)*

**Example:**
```toml
[[consensus_access_tokens]]
# The Keccak256 hash of the access token string (e.g., "ExampleAccessTokenForRPCNode").
token_hash = "0xc292251f115b36783220c56391126260cc8cb75390a062aed2879a21c1a7448f"
# The IP address of the client authorized to use this access token.
authorized_source.ipaddr = "127.0.0.1"
```

You can use the following Python code to generate the `Keccak256` hash of your access token:

```python
from Crypto.Hash import keccak
import sys

if __name__ == "__main__":
  if len(sys.argv) != 2:
    print("Usage: python keccak_string.py <string>")
    sys.exit(1)
  input_str = sys.argv[1]
  k = keccak.new(digest_bits=256)
  k.update(input_str.encode('utf-8'))
  print(f'0x{k.hexdigest()}')
```

**Usage:**
Install the `pycryptodome` library.
```sh
pip install pycryptodome
```

Then add the code to a new file called `keccak_string.py` and run the following command, replacing `YourAccessTokenHere` with you access token string.
```sh
python keccak_string.py "YourAccessTokenHere"
```
This will output the hash to use in the `token_hash` field.

### Backlog Parameters
**backlog_parameters**: (Optional) Advanced settings for managing the transaction backlog (the queue of pending transactions). Most users can use the defaults. The main options are:

- `max_backlog_size_mb` (integer): Maximum allowed backlog size in megabytes (MB). Default: 16,000 MB.
- `max_backlog_transactions_per_account` (integer): Maximum number of transactions a single account can have in the backlog at any time. Default: 25.
- `max_backlog_transaction_time_to_live_seconds` (integer): Maximum allowed time-to-live (TTL) for transactions in the backlog, in seconds. Default: 600 (10 minutes).

**Example:**
```toml
[backlog_parameters]
max_backlog_size_mb = 16000
max_backlog_transactions_per_account = 25
max_backlog_transaction_time_to_live_seconds = 600
```

### Executor Hook

**executor_hook_config**: (Optional) Configures a post-block-execution hook. When set, the node performs a configurable action (such as dumping selected database contents to disk) after executing the specified block height. All fields default to zero/absent, which disables the hook entirely.

- `block_height` (integer): Block height at which to first trigger the hook. Default: `0` (disabled).
- `interval` (integer, optional): If set, the hook repeats every `interval` blocks after the start height.
- `count` (integer, optional): If set, limits the total number of times the hook fires.
- `output_dir` (string, optional): Directory to write output to.

**Example:**
```toml
[executor_hook_config]
block_height = 5000
interval = 1000
count = 3
output_dir = "./hook_output"
```

---

### Other Parameters

**chain_state_assembler**: Advanced. Controls how the node assembles and manages the chain state. Most users should not modify this section. Options:
- `certified_block_cache_bucket_size` (integer): Number of certified blocks to keep in memory for reference to pending blocks. Default: 50.
- `sync_retry_interval_in_secs` (integer): How often (in seconds) to retry failed sync requests. Default: 1.

**Example:**
```toml
[chain_state_assembler]
certified_block_cache_bucket_size = 50
sync_retry_interval_in_secs = 1
```

**max_view_function_gas_amount**: Sets the maximum gas allowed for view (read-only) function execution. Increase or decrease to control resource usage for queries. *(integer, default: 2000000000)*

**profiling**: (Optional) Enables a profiling server for performance diagnostics. Only enable if you are debugging or profiling the node. Options:
- `enabled` (bool): Whether to enable the profiling server. Default: false.
- `host` (string): Host address to bind the profiling server to. Default: `127.0.0.1` (localhost).
- `port` (integer): Port to bind the profiling server to. Default: 9876.

**Example:**
```toml
[profiling]
enabled = true
host = "127.0.0.1"
port = 9876
```

**evm_transfer_operation_indexing**: (Optional) If `true`, enables indexing of EVM transfer operations for faster queries. *(boolean, default: false)*

**prometheus_exporter_port**: (Optional) Port for exposing Prometheus metrics for monitoring. *(integer, default: 9000)*

**max_certificates_in_backlog**: (Optional) Maximum number of transaction certificates the backlog may hold ahead of the last verified block height. For example, with a value of `1000` and a latest stored certificate height of `200`, certificates above height `1200` are rejected. *(integer, default: 1000)*

**internal_channel_capacity**: (Optional) Capacity of the internal channels that buffer synchronized data — blocks, committee authorizations and transaction-inclusion certificates — between the sync client and the node's processing pipeline. Omit to use the default. *(integer, default: 1024)*

**tokio_console_port**: (Optional) TCP port for the async-runtime `tokio-console` subscriber, bound on localhost. When omitted, an ephemeral port is chosen at startup. Set it explicitly to reach the console on a known port, and to avoid a startup port-collision race on hosts running many node processes. *(integer, optional)*

**tcp_console_port**: (Optional) TCP port for the node's `tcp_console` admin console — log-filter reload and on-demand database dump — bound on localhost. When omitted, an ephemeral port is chosen at startup. *(integer, optional)*

---

## Example

Below is a minimal example for a mainnet RPC node syncing from a validator via websocket with pruning enabled:

```toml
[chain_instance]
chain_id = 8
epoch_duration_secs = 7200
recurring_lockup_duration_secs = 172800
voting_duration_secs = 165600
is_testnet = false
genesis_timestamp_microseconds = 1732060800000000

bind_addr = "0.0.0.0:30000"
block_provider_is_trusted = false
supra_committees_config = "./configs/supra_committees.json"

[chain_state_assembler]
certified_block_cache_bucket_size = 50
sync_retry_interval_in_secs = 1

[synchronization.ws]
consensus_rpc = "ws://127.0.0.1:26000"

[synchronization.ws.certificates]
cert_path = "./configs/client_certificate.pem"
private_key_path = "./configs/client_key.pem"
root_ca_cert_path = "./configs/ca_certificate.pem"

[database_setup.dbs.archive.rocks_db]
path = "./configs/rpc_archive"
enable_pruning = true
enable_snapshots = false

[database_setup.dbs.chain_store.rocks_db]
path = "./configs/rpc_store"
enable_pruning = true
enable_snapshots = false

[database_setup.prune_config]
epochs_to_retain = 1008

[[allowed_origin]]
url = "https://rpc-mainnet.supra.com"
description = "RPC For Supra Scan and Faucet"
mode = "Server"
```
