# Profiling server

The node ships a small, optional HTTP server that exposes **memory
introspection** for a running node: jemalloc heap statistics and per-column-family
RocksDB memory usage, plus on-demand jemalloc heap profiling.

It is intended for debugging memory growth / leaks on a live node without
attaching a debugger or restarting.

## Configuration

Configured via the `[profiling]` section of the node config. It is wired into
both the validator and the RPC node configuration.

```toml
[profiling]
enabled = true        # default: false  — server does not start unless true
host = "127.0.0.1"    # default: 127.0.0.1 (localhost only, for safety)
port = 9876           # default: 9876
```

Defaults are **safe**: disabled, and bound to localhost when enabled. The server
runs on its own dedicated thread with a single-worker `ntex` runtime, so it adds
negligible overhead and cannot starve the node's main runtime.

The server is **not** feature-gated: it is present in every released build, and
`enabled` is the only gate — turning it on for a running node is a config change
plus a restart, never a different binary.

The databases that can be inspected are registered by the binary at startup:

- The validator registers `chain_db`.
- The RPC node registers `chain_db` and `archive_db`.

## Endpoints

All routes are under the `/profiling` scope.

### `GET /profiling/memory_report`

Returns JSON with jemalloc global stats and, for each registered database,
per-column-family RocksDB memory usage. No special build flags required.

```jsonc
{
  "jemalloc": {
    "allocated": 123456, "active": 130000, "metadata": 4096,
    "resident": 150000, "mapped": 200000, "retained": 50000
  },
  "databases": {
    "chain_db": {
      "total_memory_megabytes": 12.34,
      "per_column_family": [
        {
          "column_family": "blocks",
          "total_memory_megabytes": 3.21,
          "components": [
            { "property": "rocksdb.size-all-mem-tables",        "megabytes": 1.0 },
            { "property": "rocksdb.estimate-table-readers-mem",  "megabytes": 0.5 },
            { "property": "rocksdb.block-cache-usage",           "megabytes": 1.5 },
            { "property": "rocksdb.block-cache-pinned-usage",    "megabytes": 0.21 }
          ]
        }
      ]
    }
  }
}
```

Four RocksDB properties are read per column family:

- `rocksdb.size-all-mem-tables`
- `rocksdb.estimate-table-readers-mem`
- `rocksdb.block-cache-usage`
- `rocksdb.block-cache-pinned-usage`

The `default` column family is skipped (unused by the node).

The same four properties are also exported continuously as Prometheus gauges
(`supra.rocksdb.memtables_bytes.current`, `supra.rocksdb.table_readers_bytes.current`,
`supra.rocksdb.block_cache_bytes.current`, `supra.rocksdb.block_cache_pinned_bytes.current`,
labelled by `db` and `cf`) and charted in the Supra / Node Health dashboard's
RocksDB row, so reach for this endpoint when you need a point-in-time report on a
node without the metrics pipeline, and for Grafana when you want trends.

### `GET /profiling/jeprof/{command}`

On-demand jemalloc heap profiling. `{command}` is one of:

| Command       | Effect                                                                       |
| ------------- | ---------------------------------------------------------------------------- |
| `stats`       | Print full jemalloc statistics (text). No profiling build required.          |
| `activate`    | Turn runtime heap profiling **on**.                                          |
| `deactivate`  | Turn runtime heap profiling **off**.                                          |
| `reset`       | Reset accumulated profiling data.                                            |
| `dump`        | Write a heap profile to `jemalloc_outputs/jemalloc-out-<unix_ts>.heap`.      |

`stats` query parameters (only apply to `stats`):

- `len` — max output length in bytes (default 16 KiB).
- `opts` — `malloc_stats_print` option string (max 16 chars, e.g. `mdablx`).

```bash
curl 'http://127.0.0.1:9876/profiling/memory_report'
curl 'http://127.0.0.1:9876/profiling/jeprof/stats?len=65536&opts=mdablx'
curl 'http://127.0.0.1:9876/profiling/jeprof/activate'
curl 'http://127.0.0.1:9876/profiling/jeprof/dump'
```

> Every command except `stats` requires jemalloc profiling to be enabled (see
> below); otherwise the endpoint returns `400` with an explanatory message.

## Enabling jemalloc heap profiling

Heap profiling (`activate`/`deactivate`/`reset`/`dump`) needs jemalloc built
with profiling support and switched on at runtime. **The build side is already
done for you**:

1. **Build time — nothing to do.** Released binaries pin the allocator with
   jemalloc's own profiling support enabled, unconditionally and for every build
   profile, and install it as the global allocator on non-MSVC targets. There is
   no build flag to enable — a stock release build can be heap-profiled as
   shipped.

2. **Runtime — the only step.** Start the node with `MALLOC_CONF=prof:true`.
   Under systemd, via a drop-in:

   ```ini
   # /etc/systemd/system/supra-rpc.service.d/profiling.conf
   [Service]
   Environment="MALLOC_CONF=prof:true"
   ```

   followed by `systemctl daemon-reload && systemctl restart supra-rpc`. A
   drop-in is preferred over the unit's `EnvironmentFile` because it is obviously
   temporary, trivially reverted, and will not be clobbered by whatever
   provisions the host. Write that file by hand, reload and restart the unit,
   then confirm the allocator accepted the flag by calling
   `GET /profiling/jeprof/activate`.

   On some platforms (notably macOS) `MALLOC_CONF` is not reliably honoured;
   build with `JEMALLOC_SYS_WITH_MALLOC_CONF=prof:true` to bake the setting into
   the binary instead.

`GET /profiling/jeprof/stats` and `GET /profiling/memory_report` need neither
step and work on any running node; only heap *profiling* commands require the
runtime flag.

### Analyzing a dump

Analyze a dumped `.heap` file with the standard `jeprof`/`pprof` tooling:

```bash
jeprof --pdf ./supra ./jemalloc_outputs/jemalloc-out-<ts>.heap > profile.pdf
```

Symbolization happens **offline**, not on the node: a `.heap` file holds only
instruction addresses plus a `MAPPED_LIBRARIES` section (which is what lets
`jeprof` resolve load offsets for a PIE binary). So copy the small `.heap` file
off the node and resolve it against a binary elsewhere.

That binary must be the *same build* as the one that produced the dump — check
`readelf -n <binary> | grep -i 'build id'` matches on both sides, or the output
will be plausible-looking nonsense. Release assets are published in both
`-stripped` and `-unstripped` form, so fetch the `-unstripped` zip for the
deployed tag and point `jeprof` at that. Note that the stripped assets are
stripped with `objcopy --strip-debug`, which drops DWARF but keeps `.symtab`, so
the stripped binary still symbolizes to function names — the unstripped one adds
line numbers and inline-frame attribution.

## Notes & caveats

- The server has no authentication and binds to localhost by default — keep it
  that way, or front it with appropriate network controls if you bind to a
  non-loopback address.
- `memory_report` reads RocksDB properties synchronously; on a very large DB the
  call can take a moment, but it runs on the profiling server's own thread and
  does not block consensus.
