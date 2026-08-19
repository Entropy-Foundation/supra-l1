# Observability

Supra nodes emit metrics, logs, and traces so that you can tell a healthy node
from a struggling one without access to anyone else's infrastructure. This
directory documents what a node exposes and how to collect it; the ready-made
Grafana stack and dashboards live in [`observability/`](../../../observability)
at the root of this repository.

## The three signals

| Signal      | What it gives you                                   | How it leaves the node                                  |
| ----------- | --------------------------------------------------- | ------------------------------------------------------- |
| **Metrics** | Numeric time-series: heights, latencies, queue depths | Prometheus endpoint on port `9000`, scraped by you      |
| **Logs**    | Structured records of what the node did              | Rotating files on disk, tailed by a collector           |
| **Traces**  | Distributed spans showing where time went in a request | Pushed over OTLP/gRPC, only when you configure a collector |

```
                     metrics                    tracing spans / events
                        │                                │
   supra / rpc_node     ▼                                ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  Prometheus endpoint  :9000     OTLP exporter (opt-in)          │
   │  profiling server     :9876     rotating log files on disk      │
   │  (off by default)                                               │
   └────────┬───────────────────────────┬──────────────────┬─────────┘
            │ pull (scrape)             │ push (OTLP)      │ tail
            ▼                           ▼                  ▼
      Mimir / Prometheus              Tempo              Loki  ──►  Grafana
```

Three things follow from that shape, and they explain most of the setup:

- **Metrics are pull-based.** The node exposes an endpoint and waits; nothing is
  sent anywhere until you point a scraper at it. A node with no scraper is not
  misconfigured, it is just unobserved.
- **Traces are push-based and opt-in.** Nothing is exported unless you set
  `OTLP_ENDPOINT`. Leave it unset and tracing stays local, as logs.
- **Profiling is on-demand and off by default.** It answers memory questions
  when you ask, and is not part of routine monitoring.

## Where to go next

- **What ports do I open, and how do I scrape them?** → [node-telemetry.md](./node-telemetry.md)
- **Just give me a working Grafana.** → [running-the-stack.md](./running-the-stack.md)
- **What am I looking at in these dashboards?** → [dashboards.md](./dashboards.md)
- **Is a DKG round stuck?** → [dkg.md](./dkg.md)
- **This node's memory keeps growing.** → [profiling.md](./profiling.md)

## Metric naming

Every series a node emits is prefixed `supra_`, and is named
`supra_<area>_<thing>_<unit>` — for example `supra_blocks_committed_total`,
`supra_lifecycle_epoch_current`, `supra_blocks_commit_latency_seconds`. The
suffix tells you the type: `_total` is a monotonic counter, `_current` a gauge,
`_seconds` a latency histogram.

Standard process metrics (`process_resident_memory_bytes`, `process_cpu_*`,
open file descriptors, thread count) are exported on the same endpoint by both
binaries, unprefixed, following the usual Prometheus convention.

> **Counters appear only once incremented.** A counter that has never fired is
> absent from the endpoint entirely rather than reported as `0`. A healthy node
> therefore has *no series at all* for its error counters — so write alerts on
> those with `noDataState: OK`, or they will fire on healthy nodes.
