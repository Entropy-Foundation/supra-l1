# Collecting telemetry from a node

What a Supra node exposes, and how to get it into your own monitoring system.
This page assumes you run the node; it makes no assumptions about what you run
alongside it.

## Ports a node exposes

| Signal             | Port                | Default | Configurable                                                     |
| ------------------ | ------------------- | ------- | ---------------------------------------------------------------- |
| Prometheus metrics | `9000`              | on      | `prometheus_exporter_port` — `smr_settings.toml` (validator) or `config.toml` (RPC node) |
| Profiling server   | `9876`              | **off** | `[profiling]` section; localhost-only when enabled               |
| OTLP trace export  | n/a (outbound push) | off     | `OTLP_ENDPOINT` environment variable                             |

## Metrics

Both binaries install a Prometheus exporter at startup and bind it on all
interfaces at the configured port:

```toml
# smr_settings.toml (validator) or config.toml (RPC node)
prometheus_exporter_port = 9000
```

Set a different port on one of them when co-locating a validator and an RPC node
on a single host, or the two will collide on `9000`.

The endpoint is a standard Prometheus exposition on `/metrics`, so scrape it like
any other target:

```yaml
scrape_configs:
  - job_name: supra-validator
    static_configs:
      - targets: ["validator-host:9000"]
```

A 15–60s scrape interval is appropriate for production. Note that `9000` carries
no authentication and should not be reachable from the public internet — bind it
behind your firewall or a private network and scrape it from inside.

> If the exporter fails to install, the node logs an error
> ("Failed to install the global Prometheus metrics recorder …") and **keeps
> running with metrics absent**. If a node has no metrics at all, check its
> startup logs for that line before suspecting the scraper.

## Logs

The node writes structured, rotating log files under its logs directory. Point
your log collector at `<node dir>/supra_node_logs/*.log`; the bundled stack does
this with Grafana Alloy (see [running-the-stack.md](./running-the-stack.md)).

Production log levels do not emit `debug!`, and some diagnostics documented
elsewhere in this directory are only visible at `debug`. Raising the level is a
deliberate, temporary act — at mainnet committee sizes some subsystems generate
O(n²) per-message traffic that will swamp the log.

## Traces (OTLP)

Tracing is opt-in. Set `OTLP_ENDPOINT` to your collector's gRPC address before
starting the node:

```bash
export OTLP_ENDPOINT=http://otel-collector:4317
```

Use the full gRPC scheme (`http://…:4317`); a bare `host:4317` is not a valid
endpoint and the exporter will reject it. When the variable is unset, nothing is
exported and tracing stays local as logs.

Bound the volume before enabling it fleet-wide:

```bash
export OTLP_SAMPLE_RATIO=1.0    # sample everything — useful on a test node
export OTLP_SAMPLE_RATIO=0.05   # sample 5% of root traces
```

`OTLP_SAMPLE_RATIO` takes a value in `0.0`–`1.0` and defaults to `0.01`, i.e. 1%
of root traces.

> The reported `service.name` is derived from the running executable, so
> validator (`supra`) and RPC (`rpc_node`) traces can be told apart in Tempo.

## Profiling

Off by default. Enable it only while actively investigating, by adding the
section to the node config and restarting:

```toml
[profiling]
enabled = true
host = "127.0.0.1"
port = 9876
```

The server has no authentication. Leave it bound to loopback, or put real access
controls in front of it. See [profiling.md](./profiling.md) for the endpoints and
for the extra runtime flag heap profiling needs.

## A minimal production setup

If you take nothing else from this directory:

1. Scrape `:9000` on every node you run, at 15–60s.
2. Ship the node's log files somewhere you can search them.
3. Import the dashboards from [`observability/dashboards/json/`](../../../observability/dashboards/json)
   and set up alerts on epoch liveness and RocksDB write stalls at minimum — see
   [running-the-stack.md](./running-the-stack.md).
4. Leave tracing and profiling off until you have a specific question.
