# Storage

A Supra node keeps its state in RocksDB: a validator carries the ChainStore, and
an RPC node carries the ChainStore and the Archive. This directory covers the two
storage decisions an operator has to make deliberately — how much memory to give
each database, and what the compression format commits you to across an upgrade.

- [Sizing RocksDB memory](./memory-sizing.md) — choosing
  `shared_block_cache_size_mb` and `total_write_buffer_size_mb`, and reading a
  running node to tell whether the values are right
- [Upgrading past the release that enables ZSTD](./compression-upgrade.md) — why
  the v11.7.0 upgrade is one way per database, and what it means for snapshots
  and rollbacks

The fields themselves are documented in the
[node configuration guides](../node-configuration); what changed in a given
release is in the [release changelogs](../../release/mainnet).
