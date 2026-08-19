# Running the bundled observability stack

This repository ships a complete, self-contained Grafana stack under
[`observability/`](../../../observability). It is the fastest way to get real
dashboards over your own nodes, and it is a reasonable starting point for a small
production setup. If you already run Prometheus and Grafana, you do not need it —
import the dashboards and skip to [dashboards.md](./dashboards.md).

## What you get

| Service | Port                | Role                                                       |
| ------- | ------------------- | ---------------------------------------------------------- |
| Grafana | 3000                | Dashboards and alerting UI                                  |
| Alloy   | 12345, 4317, 4318   | Collector: scrapes metrics, tails logs, receives OTLP traces |
| Mimir   | 9009                | Metrics storage (Prometheus-compatible)                     |
| Loki    | 3100                | Log storage                                                 |
| Tempo   | 3200                | Trace storage                                               |

Grafana comes up pre-provisioned with the three datasources (Mimir, Loki and Tempo), all eight Supra
dashboards in a **Supra** folder, and the `supra-slo` alert rules in a
**Supra Alerts** folder.

## Prerequisites

Docker Compose or Podman Compose, and at least one running Supra node whose
metrics port you can reach from where the stack runs.

## 1. Point Alloy at your nodes

Edit [`observability/alloy/config.alloy`](../../../observability/alloy/config.alloy).
It has two blocks marked `EDIT ME`, and nothing else in it needs changing.

**Metrics targets.** One entry per node, `"<host>:<prometheus_exporter_port>"`.
The `instance` value is the name that appears in every dashboard, so choose
something you will recognise at 3am:

```
prometheus.scrape "supra_nodes" {
  targets = [
    {"__address__" = "10.0.0.11:9000", "instance" = "validator-fra-1", "role" = "validator"},
    {"__address__" = "10.0.0.12:9000", "instance" = "rpc-fra-1",       "role" = "rpc"},
  ]
  ...
}
```

If the stack runs in a container and the node runs on that same container's host,
`localhost` refers to the container, not the host. Use
`host.containers.internal` (Podman) or `host.docker.internal` (Docker).

**Log paths.** `compose.yml` mounts `observability/logs` into the collector at
`/var/log/supra`, and the default glob picks up
`/var/log/supra/*/supra_node_logs/*.log`. The simplest approach is to symlink
each node's directory into `observability/logs/`:

```bash
ln -s /opt/supra/validator observability/logs/validator-fra-1
```

Alternatively, change the bind mount in `compose.yml` to point directly at where
your nodes already write, and adjust the glob to match.

## 2. Start it

```bash
docker compose -f observability/compose.yml up -d
# or: podman-compose -f observability/compose.yml up -d
```

Grafana is then on <http://localhost:3000>, credentials `admin` / `admin`.

> Change those credentials, or keep the stack bound to a host only you can
> reach. The compose file exposes Grafana, Mimir, Loki, and Tempo on all
> interfaces with no authentication in front of them.

## 3. Check it is working

1. Open Alloy's own UI at <http://localhost:12345> and look at the components
   page. `prometheus.scrape "supra_nodes"` should show your targets as **up**. A
   target that is down is nearly always a firewall or an address that resolves
   inside the container to something other than what you meant.
2. In Grafana, open *Dashboards → Supra → Supra / Overview*. Within a minute or
   two of the first scrape you should see a block height climbing.
3. Under *Alerting → Alert rules* you should find the **Supra Alerts** folder
   with the `supra-slo` group in it.

If metrics arrive but logs do not, the mount is the usual cause: check that the
glob in `config.alloy` matches the paths *as seen inside the container*.

## 4. Tune the alerts before you rely on them

The shipped thresholds are deliberately conservative placeholders, and the Slack
webhook is a placeholder that delivers nowhere. Neither is usable as-is. See
[`observability/grafana/provisioning/alerting/README.md`](../../../observability/grafana/provisioning/alerting/README.md)
for the full rule table, what each threshold means, and how to set it from your
own observed baselines.

At minimum, before treating this as monitoring rather than dashboards:

- Set the **epoch not advancing** lookback window above your network's longest
  expected epoch, or it will fire during normal operation.
- Replace the Slack webhook URL, or wire up a contact point of your own.

## 5. Optional: traces

Nothing pushes traces until you ask a node to. Start the node with the stack's
Alloy as its collector:

```bash
export OTLP_ENDPOINT=http://<alloy-host>:4317
export OTLP_SAMPLE_RATIO=0.05
```

See [node-telemetry.md](./node-telemetry.md) for what those control.

## Stopping

```bash
docker compose -f observability/compose.yml down          # keep stored data
docker compose -f observability/compose.yml down -v       # discard it too
```
