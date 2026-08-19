# `rpc_node` CLI Change Log

Changes to the `rpc_node` command-line tool since `supra_node_v11.3.6`.

---

## New Commands

### `rpc_node export table`

Extracts one or more tables (RocksDB column families) from a database directory into compact per-table binary dump (`.stbd`) files that can be copied between machines and diffed. The source database is opened read-only, so the command works directly on a post-execution-hook checkpoint while the source node is still running, as well as on a stopped node's database. On an rpc node this covers both the chain-store and archive databases.

An `.stbd` file stores the raw key/value bytes exactly as they live in RocksDB, in RocksDB key order, zstd-compressed, with an entry count and CRC32 checksum so truncation or corruption is detected on read.

| Flag | Description |
|---|---|
| `--db-path <PATH>` | Path to the source RocksDB directory (a checkpoint dir or a stopped node DB). Required. |
| `--table-names <NAMES>` | Table names to extract; comma-separated and/or repeated. Unknown names fail up-front with the list of available tables. Required. |
| `--out-dir <DIR>` | Directory where the `<table>_<label>.stbd` files are written. Required. |
| `--label <LABEL>` | Optional free-text label appended to each output filename and stored in the dump header (e.g. a block height or node name). |
| `--start-key <HEX>` | Start key for iteration (inclusive, hex-encoded), same convention as `export raw`. |
| `--end-key <HEX>` | End key for iteration (exclusive, hex-encoded). |

### `rpc_node export table-diff`

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
