# Storage Change Log

Changes to on-disk storage since `supra_node_v11.5.1`.

This file records changes to the node's database format and to what an operator can do with a
database or a snapshot.

---

## Changed Behavior

### ZSTD compression makes the upgrade one-way per database

**Breaking change.** The history-bearing column families of the ChainStore and the Archive are now
compressed with LZ4 on the upper levels and ZSTD on the bottommost level, where the bulk of the
bytes settle.

RocksDB's compression codecs are linked at build time, and a binary can read only the codecs it
links. No previously released binary links ZSTD — every `supra_node_v11.5.x` release was built with
LZ4 and Snappy only — and this release does. Reads are therefore compatible in one direction only.

**A database written by this release cannot be opened by an earlier binary.** From the first
bottommost compaction onwards — which happens on RocksDB's own schedule some time after the node
starts, not at the upgrade itself — an earlier binary pointed at the same data directory starts
normally and then fails every read that touches a ZSTD block. The affected column families include
`transactions` and `tx_output`, which the transaction read APIs are served from, so there is no
partial-service mode.

**A snapshot taken from an upgraded node cannot be restored onto an earlier binary either.** A
snapshot is a RocksDB checkpoint of the live SST files, so it carries the same blocks. This is the
wider of the two constraints, because snapshot restore is the normal way a node is bootstrapped
rather than something reached for only in an incident.

**Operator action.**

- Treat the upgrade as irreversible for that database from the moment the new binary opens it.
- The Foundation will maintain up-to-date old-format snapshots until sufficiently many operators
  have adopted the new binary. In the event that a rollback is required, follow the standard
  snapshot-restore workflow.
- When bootstrapping a node from a post-upgrade snapshot, install this release's binary **first**,
  then restore.
- Compression cannot be disabled from configuration. The policy is chosen per column family in
  code and is deliberately absent from `smr_settings.toml` and `config.toml`.

Nodes already running this release, and snapshots exchanged between them, are unaffected: the new
binary reads everything an earlier one wrote.

See [Upgrading past the release that enables ZSTD](../../../operations/storage/compression-upgrade.md)
for the runbook.
