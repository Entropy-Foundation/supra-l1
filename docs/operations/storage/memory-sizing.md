# Sizing RocksDB memory for a Supra node

How to choose `shared_block_cache_size_mb` and `total_write_buffer_size_mb` for a node, and how to
tell from a running node whether the values you chose are right. Both settings were introduced in
v11.7.0; see that release's [configuration changelog](../../release/mainnet/v11.7.0/config.md) for
the exact config-file placement on validators and RPC nodes, and the
[node configuration guides](../node-configuration) for the current field reference.

## The one thing to know first

Each database has **one** LRU cache, and it holds two unrelated things:

- **Read blocks** — data, index and filter blocks, which is what a block cache normally holds.
- **Memtables** — the `WriteBufferManager` reserves memtable memory *out of that same cache*.

So `shared_block_cache_size_mb` is the whole memory envelope for a database, and
`total_write_buffer_size_mb` is the share of it that writes are allowed to claim. They are not
independent numbers, and the second must never exceed the first. A node whose configuration
violates that refuses to start.

The practical consequence: if the memtable cap is close to the cache size, a write burst can evict
the entire read cache, and reads that were being served from memory start going to disk exactly
when the node is busiest.

## Defaults

| Database | Cache | Memtable cap | Write share |
|---|---|---|---|
| ChainStore | 2048 MiB | 1024 MiB | 50% |
| Archive | 8192 MiB | 2048 MiB | 25% |

A validator carries only the ChainStore, so its expected RocksDB footprint is ~2 GiB. An RPC node
carries both, so ~10 GiB. Omit both fields to get these; they are sized for the ~30 GB hosts the
Foundation's nodes run on.

## Choosing values for a different host

1. **Start from the read side.** The cache has to hold the index blocks of the SSTs the node
   actually reads, before it can cache any data. On a full archive this is the dominant term: with
   4 KiB blocks over ~60 MB SSTs, a single file's index runs to hundreds of KB, and large column
   families have been measured at 62–96% index blocks with as little as 0% data blocks. Shrinking
   the archive cache does not shrink that requirement; it just removes the data caching.
2. **Then pick the write share.** Keep the memtable cap at or below half the cache for the
   ChainStore and a quarter for the Archive. Going higher trades read caching for larger flushes.
3. **Leave the host room for page cache.** RocksDB reads that miss the block cache are served from
   the OS page cache, so a node sized to consume all available RAM performs worse than one that
   leaves headroom, not better. Budget the RocksDB envelope, the process heap outside RocksDB
   (~1 GiB) and thread stacks, and leave the remainder free.

On a host smaller than ~16 GB, lower the **Archive** cache first — it is by far the largest term —
and lower its memtable cap in step so the write share stays near a quarter.

## Checking a running node

- **Is the memtable cap binding?** Look for `flush_reason: "Write Buffer Manager"` in the RocksDB
  info LOG. A flush attributed to the write buffer manager is one the cap forced early, before the
  column family reached its own `write_buffer_size`. Occasional entries under peak load are the
  safety net working. Entries dominating the log mean the cap is too low for the write rate, and
  the cost is small SST files and the extra compaction they generate.
- **How much of each budget is actually used?** The profiling server's memory report (see
  [../observability/profiling.md](../observability/profiling.md)) reports `block-cache-usage` once
  per database, under `shared_block_cache`, and `size-all-mem-tables` per column family. Usage
  sitting far below capacity means the budget can be reduced; usage pinned at capacity with poor
  read latency means the cache is too small.

  Read the report's `total_memory_megabytes` as the whole envelope and do **not** add the memtable
  figures to it: the write-buffer manager charges memtable memory into the shared cache, so it is
  already inside `block-cache-usage`. The per-column-family memtable numbers are there to show the
  distribution — which families are holding the memory — not to be summed.
- **What is in the cache?** The periodic `Block cache entry stats` lines in the info LOG break usage
  down by block type. A family reporting a high `IndexBlock` share and a near-zero `DataBlock` share
  is caching only metadata, and giving it a larger cache buys data caching rather than more of the
  same.

## Two things not to do

- **Do not set either field to `0`.** RocksDB reads a zero as "not configured" rather than
  "disabled": a zero memtable cap leaves total memtable memory unbounded, and a zero cache capacity
  falls back to a small per-family default. Both are the opposite of what the setting reads as, so
  the node refuses to start.
- **Do not expect the memtable cap to bound the WAL.** A WAL file can only be recycled once every
  memtable holding its writes has flushed, so a larger cap defers those flushes and raises the
  steady-state WAL floor. The floor is capped separately, at 8 GiB per database, and that cap is
  also the volume a restart replays (about half a minute per database) and the checkpoint threshold
  the snapshot service uses. When the live WAL reaches it, RocksDB flushes every column family with
  unflushed data — the same flush a snapshot checkpoint performs. Look for
  `Flushing all column families with data in WAL number` in the info LOG to see it firing.
