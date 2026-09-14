# Upgrading past the release that enables ZSTD

The release described by [v11.7.0's storage changelog](../../release/mainnet/v11.7.0/storage.md)
compresses the history-bearing column families of the ChainStore and the Archive with ZSTD on their
bottommost level. The compression win is worth having, but enabling it is a **one-way door for the
database**: once a node has run this release for long enough to compact, its database can no longer
be opened by an earlier binary, and neither can any snapshot taken from it.

This runbook says exactly what that constrains, what the failure looks like if it is ignored, and
what to do instead.

## Why it is one-way

RocksDB's compression codecs are compile-time options of the native library. A binary links the
codecs it was built with and can read only those; a block written with a codec the binary does not
link fails to decompress with
`Status::NotSupported("Unsupported compression method for this build")`. There is no fallback path
and no way to decompress the block later.

No previously released binary was built with ZSTD; every binary from this release onwards links it.
So the incompatibility runs in one direction only: a new binary reads everything an old one wrote,
and an old binary cannot read what the new one writes to the bottommost level.

The affected column families include `transactions` and `tx_output`, which every transaction read
API is served from, so there is no partial-service mode: an old binary pointed at an upgraded
database fails the reads that matter rather than degrading.

## What this constrains

### Rolling back to an earlier binary

**Stopping the node and starting the previous binary on the same data directory is not a rollback
path after the upgrade.** Nothing warns you at the point of no return, and the point of no return
is not the upgrade itself — it is the first bottommost compaction, which happens on RocksDB's own
schedule some time after the node starts serving. A rollback attempted in the first minutes may
appear to work and the same rollback attempted an hour later may not.

Treat the upgrade as irreversible from the moment the new binary opens the database.

### Restoring a snapshot onto an earlier binary

A node snapshot is a RocksDB checkpoint — the live SST files as they are on disk — so a snapshot
taken from an upgraded node carries ZSTD blocks and can only be restored onto a binary from this
release or later.

In particular: **a fresh node bootstrapped from a post-upgrade snapshot must be running the new
binary before it is given the snapshot.** Upgrade the binary first, then restore.

### Turning the compression off

It cannot be turned off from configuration. The compression policy is chosen per column family in
the binary and is deliberately absent from `smr_settings.toml` and `config.toml` — an
operator-settable value here would let two nodes disagree about what their databases contain.
Changing it needs a new binary, and even then, a build without ZSTD linked would not be able to
read the blocks already written.

## If you need to go back

The only supported route is to start the older binary on a database that has never been opened by
the new one. The Foundation will continue to provide access to up-to-date old-format snapshots until
sufficiently many operators have adopted v11.7.0.

## What it looks like if it is ignored

An older binary opening an upgraded database **starts normally**. RocksDB validates the compression
configuration at open, but the read path only discovers an unsupported codec when it first has to
decompress a block, so the failure surfaces later as read errors on the affected column families —
transaction and block queries failing while the node otherwise looks healthy.

The corresponding *build* mistake — a binary that selects ZSTD but was not built with it — is
caught at startup. Every compression type the tuning layer can select is probed when the first
database is opened, and a type the build does not link is reported as a refusal to start, naming
the type. RocksDB's own validation does not cover `bottommost_compression`, which is why this check
exists.
