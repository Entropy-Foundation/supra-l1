# Node Configuration Change Log

Changes to node configuration files since `supra_node_v11.5.1`.

---

## `smr_settings.toml` — Validator Node Settings

### Changed validation: `node.ws_server.certificates.root_ca_cert_path` is now required

**Breaking change.** The CA certificate that anchors the validator-RPC mTLS channel must be
configured. It is the trust anchor the validator authenticates connecting RPC nodes against, and
only the certificates issued by it are now admitted on that channel.

A validator that omits the field, or sets it to an empty or whitespace-only string, no longer
starts. Startup fails before the WebSocket server is bound, with an error naming
`node.ws_server.certificates` and the `root_ca_cert_path` field.

```toml
[node.ws_server.certificates]
# Required. The CA whose certificates this validator will accept from RPC nodes — for the common
# case of a validator acting as its own CA, this is the CA certificate it issues those with.
root_ca_cert_path = "/node/ca_certificate.pem"
cert_path         = "/node/server_supra_certificate.pem"
private_key_path  = "/node/server_supra_key.pem"
```

**Operator action.** Set `root_ca_cert_path` before upgrading. A configuration that was legal on
`supra_node_v11.5.1` and omits it will fail to start on this release. Validators that already set
it are unaffected, and no certificate needs reissuing.

The settings template written by `supra node smr-settings` now includes the key, with an empty
value alongside the `cert_path` and `private_key_path` placeholders it already emitted. Previously
it omitted the key entirely, which would have left a generated template failing to start on a
field the operator never saw.

### New field: `node.ws_server.allowed_client_certificate_hashes`

Optional list of the RPC-node certificates this validator will serve, each the hex-encoded
Keccak-256 digest of the certificate's DER encoding — the same value the node already logs when it
accepts a connection (`Accepted connection from …; certificate hash …`).

Keccak-256 is not the SHA-256 fingerprint `openssl x509 -fingerprint -sha256` prints. That
fingerprint is 32 bytes too, so it parses and the validator starts — and then refuses every RPC
node. Copy the digest from the log line, or compute it with
`openssl x509 -in <cert> -outform der | keccak-256sum`.

Chaining to `root_ca_cert_path` establishes that the operator's CA issued a certificate, not that
it was issued for this channel. Listing the certificates makes the channel's membership a closed
set instead: a client whose certificate is not listed is refused after the handshake,
with the hash it presented named in the log so the operator can add it if the refusal was a
mistake.

```toml
[node.ws_server]
allowed_client_certificate_hashes = [
  "0x9f2c…",  # rpc-1
  "0x41ab…",  # rpc-2
]
```

| Field | Type | Default |
|---|---|---|
| `node.ws_server.allowed_client_certificate_hashes` | array of hex strings | unset |

When the field is omitted — the default, and the behaviour of every earlier release — any
certificate the configured CA issued is admitted, as before. Two values fail startup rather than
being accepted quietly:

- An entry that is not a hex-encoded 32-byte digest, named in the error. Skipping it would silently
  narrow the list and lock out the node it was meant to name. Both a bare and a `0x`-prefixed digest
  are accepted, and surrounding whitespace is ignored.
- An empty list (`allowed_client_certificate_hashes = []`). Since omitting the field admits every
  certificate, an empty list is the one value whose meaning inverts what was written — it would
  refuse every RPC node. Omit the field instead.

Recommended for validators serving a known, stable set of RPC nodes. Note that the list must be
updated when an RPC node's certificate is reissued or rotated, and that a node restart is
required for a change to take effect.

### New fields: `node.ws_server.max_concurrent_tls_handshakes`, `node.ws_server.max_concurrent_tls_handshakes_per_source`, `node.ws_server.tls_handshake_timeout_secs`, `node.ws_server.websocket_upgrade_timeout_secs`

Bound the validator-RPC accept path against a client that opens a connection and then stalls it,
deliberately or otherwise.

```toml
[node.ws_server]
max_concurrent_tls_handshakes            = 256
max_concurrent_tls_handshakes_per_source = 8
tls_handshake_timeout_secs               = 10
websocket_upgrade_timeout_secs           = 10
```

| Field | Type | Default |
|---|---|---|
| `node.ws_server.max_concurrent_tls_handshakes` | integer | `256` |
| `node.ws_server.max_concurrent_tls_handshakes_per_source` | integer | `8` |
| `node.ws_server.tls_handshake_timeout_secs` | integer (seconds) | `10` |
| `node.ws_server.websocket_upgrade_timeout_secs` | integer (seconds) | `10` |

`max_concurrent_tls_handshakes` bounds how many TLS handshakes the server runs at once; a
connection accepted past that limit is dropped immediately rather than queued.
`max_concurrent_tls_handshakes_per_source` bounds how many of those any one source address may
hold, so a single host cannot take the whole budget and starve the rest.
`tls_handshake_timeout_secs` bounds how long any one handshake may take before the connection
holding it is dropped. `websocket_upgrade_timeout_secs` bounds what follows a completed handshake
— reading the client's HTTP request headers and writing the response that hands the connection
over to the WebSocket. The two timeouts apply to one accept path, so a connection has at most
`tls_handshake_timeout_secs + websocket_upgrade_timeout_secs` from being accepted to holding a
live WebSocket, and whatever the handshake leaves unspent is still available to the upgrade.

**These defaults introduce limits that did not previously exist.** All four fields are optional;
when omitted the defaults above apply, whereas earlier releases bounded none of them. A validator
that legitimately sees more than 256 handshakes at once — a mass certificate rotation, say — or
more than 8 from a single address, or a slow mTLS path that needs longer than 10 seconds, will see
new refusals after upgrading and should raise the corresponding field. Each must be at least `1`;
a `0` is refused at startup, since it would refuse every connection rather than lift the limit.
Two further values are refused at startup rather than accepted as written: either concurrency
bound above `1048576` or either timeout above `86400` seconds, and a
`max_concurrent_tls_handshakes_per_source` that you set explicitly at or above
`max_concurrent_tls_handshakes` — a source's share has to be smaller than the budget it is a share
of, since at or above it the per-source bound can never refuse a connection. Lowering
`max_concurrent_tls_handshakes` on its own is always safe: where the per-source field is unset, its
default is fitted to just below the budget rather than refused, so the per-source bound keeps
working. A node restart is required for a change to
take effect.

### New section: `node.database_setup.compaction_config`

Settings of a new scheduled maintenance pass that merges the small RocksDB column families of
every database the node opens. Leveled compaction never merges the bottom level of
a family that fits in its base level, so a family that is written a little at a time accumulates
one SST file per flush indefinitely; each of those files costs a manifest entry, an open file
handle, an index block in the block cache and a seek when an iterator is built. The pass rewrites
those families with an explicit full-range compaction, which is the only thing that merges them.

```toml
[node.database_setup.compaction_config]
enable_scheduled_compaction = true
interval_in_hours           = 24
max_family_size_in_mib      = 15360
min_live_files              = 64
```

| Field | Type | Default |
|---|---|---|
| `enable_scheduled_compaction` | boolean | `true` |
| `interval_in_hours` | integer (hours) | `24` |
| `max_family_size_in_mib` | integer (MiB) | `15360` (15 GiB) |
| `min_live_files` | integer | `64` |

**The pass runs by default.** The whole section is optional and every field within it is
optional, so an existing configuration file keeps working unchanged — and gains the pass, since
that is what fixes the file-count backlog on a node that already has one. Set
`enable_scheduled_compaction = false` to opt out.

A family is rewritten only when it has at least `min_live_files` live SST files *and* its total
SST size is below `max_family_size_in_mib`. Larger families compact organically and are not where
the file count is, so rewriting them would cost far more I/O than the saving is worth; families
that are already tidy are left alone rather than rewritten every day for nothing. The deprecated
`prune_index` family is never rewritten — it is being retired instead.

The pass never runs at the same time as pruning: both take a single process-wide maintenance slot.
Pruning is the job with a deadline, so it waits for the slot rather than skipping, and the
compaction pass gives way between column families as soon as a prune starts waiting — a prune is
therefore delayed by at most the one family being rewritten, and resumes the rest of the pass at
the next interval.

The first pass runs an hour after startup (or after the configured interval, if that is shorter),
not immediately: a node that has just started is catching up and its caches are cold, but a full
interval would mean a node restarted more often than the interval never compacts at all.

Each field must be at least `1` when the pass is enabled; a `0` is rejected at startup with an
error naming the field. A node restart is required for a change to take effect.

---

### New field: `node.database_setup.max_background_jobs`

Optional override for the size of RocksDB's background-job pool — the threads that run memtable
flushes and compactions. Omit it to derive the value from the host CPU count, which is the
recommended setting.

```toml
[node.database_setup]
# Optional. Number of RocksDB background threads (flush + compaction). Must be at least 1.
# Omit to derive from the host CPU count.
max_background_jobs = 6
```

RocksDB's own default is 2, which is too few for the database sizes these nodes now reach, so the
derived value is used when the field is absent.

The setting is deliberately **process-wide** rather than per storage instance. RocksDB's background
threads belong to the environment that the node's databases share, and opening a database can only
raise that pool's ceiling — never lower it, and never add to it. A per-instance setting would
therefore have given every database the largest value any of them asked for, so the field sits
next to `prune_config` and `snapshot_config` rather than under `dbs.<instance>.rocks_db`.

A value below 1 is rejected at startup. RocksDB derives its limits as
`max_flushes = max(1, jobs / 4)` and `max_compactions = max(1, jobs - max_flushes)`, so a
non-positive value lands on one flush and one compaction thread — the same as its stock default of
2 — meaning the setting cannot express what a zero or negative value implies. The node therefore
fails to start with an error naming the field rather than silently ignoring it.

### New fields: `node.database_setup.dbs.chain_store.rocks_db.shared_block_cache_size_mb` and `total_write_buffer_size_mb`

Optional per-database memory budgets, both in mebibytes. Omit either to use the
built-in default.

```toml
[node.database_setup.dbs.chain_store.rocks_db]
path = "store"
# Optional. Size of the single block cache shared by every column family of this database.
# Omit for the built-in default (2048 MiB for the ChainStore, 8192 MiB for the archive).
shared_block_cache_size_mb = 2048
# Optional. Cap on the total memory held by all of this database's memtables, charged to the
# same shared cache. Omit for the built-in default (1024 MiB for the ChainStore, 2048 MiB for
# the archive).
total_write_buffer_size_mb = 1024
```

Unlike `max_background_jobs`, these are **per database**: the block cache and the write-buffer
manager are objects owned by the database, so each instance genuinely receives the budget
configured for it. Previously every column family created its own cache, so the effective read
cache was the sum of two dozen independent caches and was neither observable nor bounded as a
whole.

Setting either field to `0` is rejected at startup. RocksDB reads a zero there as "not configured"
rather than "disabled" — a zero memtable cap leaves the total unbounded, which is the opposite of
what the setting reads as — so the node fails to start with an error naming the field.

`total_write_buffer_size_mb` must also not exceed `shared_block_cache_size_mb`, and is checked
against the built-in default when only one of the two is set. Memtable memory is charged to the
same shared cache, so a larger cap would let memtables evict the whole read cache and the cache
size would stop bounding the database's combined footprint.

The defaults differ per database, and are sized against measured usage on a mainnet node rather
than estimated:

| Database | `shared_block_cache_size_mb` | `total_write_buffer_size_mb` | Write share |
|---|---|---|---|
| ChainStore | `2048` | `1024` | 50% |
| Archive | `8192` | `2048` | 25% |

A validator carries only the ChainStore, so its expected RocksDB footprint is ~2 GiB; an RPC node
carries both, so ~10 GiB. Both are sized for the ~30 GB hosts the network runs on.

**Read-cache capacity is roughly unchanged, but expect the memory to actually be used.** The
previous release gave every column family its own unshared cache, totalling about 9.6 GiB per RPC
node, so the new ~10 GiB envelope is about the same nominal read capacity. Two things did change:
memtable memory is now bounded and counted *inside* that envelope rather than being unbounded
alongside it, and one shared cache on a full archive will fill to capacity where two dozen
fragmented caches did not. Operators on 30 GB hosts should therefore revisit these values rather
than assume the upgrade reduces the footprint.

**Operators on smaller hosts should read
[Sizing RocksDB memory](../../../operations/storage/memory-sizing.md)**, which
covers how to choose values for a given amount of RAM, how to tell from a running node whether the
memtable cap is binding, and why raising the cap interacts with WAL retention.

---

### Removed field: `table.<name>.l2_cache`

**Breaking change.** The per-table `l2_cache` size is no longer accepted anywhere a table can be
configured, on either node type. Every column family now draws from its database's
single shared block cache, sized by `shared_block_cache_size_mb`, so a per-table native-cache size
no longer describes anything the storage engine does.

A configuration that still sets it fails to start, with an error naming the table and the field:

```
table `batch` sets `l2_cache`, which is no longer supported: every column family shares one
block cache sized by `shared_block_cache_size_mb`. Remove `l2_cache` from the table configuration.
```

**Operator action.** Remove every `l2_cache` line from the `[*.database_setup.dbs.*.rocks_db.table.*]`
sections before upgrading. `l1_cache` and `prunable` are unchanged and still accepted. Nodes with no
`table` section — the common case — need no change, and generated configuration files no longer
contain the field.

It is rejected rather than ignored so that a value an operator believed was taking effect cannot
silently do nothing.

---

## `config.toml` — RPC Node Settings

### Removed field: `table.<name>.l2_cache`

**Breaking change.** The per-table `l2_cache` size is no longer accepted anywhere a table can be
configured, on either node type. Every column family now draws from its database's
single shared block cache, sized by `shared_block_cache_size_mb`, so a per-table native-cache size
no longer describes anything the storage engine does.

A configuration that still sets it fails to start, with an error naming the table and the field:

```
table `batch` sets `l2_cache`, which is no longer supported: every column family shares one
block cache sized by `shared_block_cache_size_mb`. Remove `l2_cache` from the table configuration.
```

**Operator action.** Remove every `l2_cache` line from the `[*.database_setup.dbs.*.rocks_db.table.*]`
sections before upgrading. `l1_cache` and `prunable` are unchanged and still accepted. Nodes with no
`table` section — the common case — need no change, and generated configuration files no longer
contain the field.

It is rejected rather than ignored so that a value an operator believed was taking effect cannot
silently do nothing.

---

### New fields: `database_setup.dbs.<instance>.rocks_db.shared_block_cache_size_mb` and `total_write_buffer_size_mb`

The same optional per-database memory budgets as on the validator, with the same rejection of `0`
and the same requirement that `total_write_buffer_size_mb` not exceed
`shared_block_cache_size_mb`. An RPC node has two instances, so each can be budgeted separately:

```toml
[database_setup.dbs.chain_store.rocks_db]
path = "rpc_store"
shared_block_cache_size_mb = 2048
total_write_buffer_size_mb = 1024

[database_setup.dbs.archive.rocks_db]
path = "rpc_archive"
# The archive serves the read APIs over a much larger index, so its budgets are larger.
shared_block_cache_size_mb = 8192
total_write_buffer_size_mb = 2048
```

---

### New field: `database_setup.max_background_jobs`

The same optional override as on the validator, with the same process-wide semantics and the same
rejection of values below 1. Omit it to derive the value from the host CPU count.

```toml
[database_setup]
# Optional. Number of RocksDB background threads (flush + compaction). Must be at least 1.
# Omit to derive from the host CPU count.
max_background_jobs = 6
```

On an RPC node the pool is shared by the ChainStore and the archive, so one value covers both.

---

### Changed validation: `synchronization.ws.certificates.root_ca_cert_path` is now required

**Breaking change.** The same requirement on the other end of the channel: an RPC node that
synchronizes over WebSocket must configure the CA certificate of the validator it syncs from,
which is the only authority whose certificates it will accept there.

An RPC node that omits the field, or sets it to an empty or whitespace-only string, no longer
starts. The config is rejected during the startup validation that runs before any database is
opened, with an error naming `synchronization.ws.certificates` and the `root_ca_cert_path` field.
Nodes synchronizing over REST are not subject to the check.

```toml
[synchronization.ws.certificates]
# Required. Ask the operator of the validator you sync from for its CA certificate; it is the
# same CA that issued your node's `cert_path`.
root_ca_cert_path = "./configs/ca_certificate.pem"
cert_path         = "./configs/client_certificate.pem"
private_key_path  = "./configs/client_key.pem"
```

**Operator action.** Set `root_ca_cert_path` before upgrading. Nodes synchronizing over REST
(`synchronization.rest`) are unaffected, as are RPC nodes that already set the field.

### New section: `database_setup.compaction_config`

The same scheduled compaction pass and the same fields as
`node.database_setup.compaction_config` above, applied to both databases an RPC node opens
(the chain store and the archive). The full-history RPC node is where the tiny-SST backlog is
largest, so this is the node type the pass matters most for.

```toml
[database_setup.compaction_config]
enable_scheduled_compaction = true
interval_in_hours           = 24
max_family_size_in_mib      = 15360
min_live_files              = 64
```
