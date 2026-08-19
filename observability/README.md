# Observability assets

A ready-to-run Grafana stack and the Supra dashboards, for operators monitoring
their own validator and RPC nodes.

**Start here:** [Running the bundled stack](../docs/operations/observability/running-the-stack.md)
walks through pointing this at your nodes. The rest of the observability
documentation is in [`docs/operations/observability/`](../docs/operations/observability).

## What's here

| Path                             | What it is                                                        |
| -------------------------------- | ------------------------------------------------------------------ |
| `compose.yml`                    | The stack: Grafana, Mimir, Loki, Tempo, Alloy                     |
| `alloy/config.alloy`             | Collector config — **edit this to name your nodes**               |
| `dashboards/json/`               | Eight Supra dashboards, importable into any Grafana               |
| `grafana/provisioning/`          | Datasources, dashboard provisioning, and the `supra-slo` alert rules |
| `mimir.yml`, `loki.yml`, `tempo.yml` | Backend configuration; no changes normally needed             |
| `logs/`                          | Mount point for your nodes' log directories                       |

## Using the dashboards on their own

You do not need this stack. The files in `dashboards/json/` are plain Grafana
dashboards — import them into an existing Grafana and point them at any
Prometheus-compatible datasource holding your nodes' metrics. They expect the
datasource to be selectable via a `datasource` variable and filter on an
`instance` label.

## Before exposing any of this

The compose file ships development-grade defaults: Grafana's `admin`/`admin`
credentials, and every backend published on all interfaces with no
authentication. It is safe on a workstation and is **not** safe on a public
network. Change the credentials and restrict the ports, or run it behind an
authenticating proxy.

The alert rules ship with placeholder thresholds and a placeholder Slack webhook
that delivers nowhere. See
[`grafana/provisioning/alerting/README.md`](./grafana/provisioning/alerting/README.md)
before relying on them.
