# Supra CLI Change Log

Changes to the `supra` command-line tool since `supra_node_v11.3.6`.

---

## New Commands

### `supra data export table`

Extracts one or more tables (RocksDB column families) from a database directory into compact per-table binary dump (`.stbd`) files that can be copied between machines and diffed. The source database is opened read-only, so the command works directly on a post-execution-hook checkpoint while the source node is still running, as well as on a stopped node's database.

An `.stbd` file stores the raw key/value bytes exactly as they live in RocksDB, in RocksDB key order, zstd-compressed, with an entry count and CRC32 checksum so truncation or corruption is detected on read.

| Flag | Description |
|---|---|
| `--db-path <PATH>` | Path to the source RocksDB directory (a checkpoint dir or a stopped node DB). Required. |
| `--table-names <NAMES>` | Table names to extract; comma-separated and/or repeated. Unknown names fail up-front with the list of available tables. Required. |
| `--out-dir <DIR>` | Directory where the `<table>_<label>.stbd` files are written. Required. |
| `--label <LABEL>` | Optional free-text label appended to each output filename and stored in the dump header (e.g. a block height or node name). |
| `--start-key <HEX>` | Start key for iteration (inclusive, hex-encoded), same convention as `data export raw`. |
| `--end-key <HEX>` | End key for iteration (exclusive, hex-encoded). |

### `supra data export table-diff`

Diffs two tables by raw key bytes via a streaming merge-join, classifying every key as left-only, right-only, or a value mismatch. Each side is either an `.stbd` dump file or a database directory (a DB side requires `--table`), so dump-vs-dump, dump-vs-DB, and DB-vs-DB comparisons are all supported. Reports are written under `<out-dir>/<table>/` as `summary.json` plus per-category `left_only.json` / `right_only.json` / `mismatch.json`, with entries decoded to readable JSON where a decoder is registered (hex fallback otherwise).

| Flag | Description |
|---|---|
| `--left <PATH>` | Left side: an `.stbd` dump file or a DB directory. Required. |
| `--right <PATH>` | Right side: an `.stbd` dump file or a DB directory. Required. |
| `--table <NAME>` | Table name. Required when either side is a DB directory; for two dumps it overrides the header table names, which must otherwise match. |
| `--out-dir <DIR>` | Directory where the diff report files are written. Required. |
| `--concise` | Emit counts only; no per-entry detail files. |
| `--limit <N>` | Cap the number of per-entry differences written to the detail files. Summary counts always reflect the full scan. |
| `--start-key <HEX>` | Start key (inclusive, hex-encoded) applied to both sides. |
| `--end-key <HEX>` | End key (exclusive, hex-encoded) applied to both sides. |

These flags compose into an end-to-end workflow: take a height-consistent checkpoint with the post-execution hook, extract the tables of interest, then diff them across machines.

---

## Changed Behavior

### `supra config generate` — new `--genesis-mode` flag

This command is for local testing only; the flag does not affect any production network.

Supra's testnet and mainnet were bootstrapped by two different genesis encoders, and the mainnet one publishes fewer resources — it omits the core-resources account and its mint capability, the config buffer, the DKG state, the reconfiguration-in-progress indicator, the randomness resources, the leader-ban config, the JWK/keyless resources, and the EVM genesis config. Previously a generated local network always used the testnet encoder, so an upgrade whose scripts have to publish one of those resources could be verified only against a network that already had it.

`--genesis-mode <testnet|mainnet>` selects the encoder. It defaults to `testnet`, which is the previous behavior, so existing invocations are unaffected. A `mainnet` network still has its full validator set and a working faucet, but anything requiring the core-resources account (most visibly `version::set_version`) and the EVM will not work on it.


| Flag | Default | Description |
|---|---|---|
| `--genesis-mode <MODE>` | `testnet` | Genesis specification to initialise the network with: `testnet` or `mainnet`. |

### `supra move account rotate-key`

Rotation is now crash-safe. The new key is durably written to `cli_data.db` as a *staged* authentication key **before** the transaction is submitted; only after the transaction finalizes as `Success` is it atomically promoted to the active key and cleared. A run interrupted between submit and promotion leaves the profile recoverable rather than locked out.

Recovery is automatic. When a staged key is present, the next `rotate-key` reconciles it by comparing the account's current on-chain authentication key against the staged key: if they already match, the rotation landed and the staged key is adopted as the active key; if they differ but the staged rotation's recorded expiration has not yet passed (per the on-chain clock), the command reports that a rotation is already in progress and exits without starting a competing one; if they differ and that expiration has passed, the stale staged key is discarded and a fresh rotation proceeds. If the node is unreachable, the staged key is left untouched so a later run can reconcile it.

**Breaking change.** The environment variable that supplies the new key was renamed from `CLI_PROFILE_AUTHENTICATION_KEY` to `CLI_PROFILE_STAGED_AUTH_KEY`. Scripts that set the old variable for `rotate-key` must update to the new name. `CLI_PROFILE_AUTHENTICATION_KEY` continues to be read by `supra profile new`.

Staging the key required a `cli_data.db` schema change, applied automatically by migration `0002` the first time this build opens an existing database: the `pending_authentication_private_key` column is renamed to `staged_authentication_private_key`, and a `staged_expiration_timestamp_secs` column is added to record when the staged rotation transaction expires. No operator action is required.

### Transaction finalization wait

Commands that submit a transaction (`transfer`, `publish`, governance, multisig, automation, `node identity` rotations, `move account rotate-key`, …) now decide how long to wait for finalization using the chain's on-chain clock rather than the local clock. The CLI keeps waiting until the transaction's expiration passes on-chain, and no longer hangs when the node is unreachable: it stops waiting a short margin past the expiration by the local clock and returns an error. While waiting it prints a periodic warning to stderr if the chain's timestamp stops advancing or the node cannot be reached. Scripts that capture stderr or rely on the previous wait timing may observe these differences.
