# Reading the dashboards

Eight Grafana dashboards ship in
[`observability/dashboards/json/`](../../../observability/dashboards/json). They
are provisioned automatically by the bundled stack
([running-the-stack.md](./running-the-stack.md)); to use them with an existing
Grafana, import the JSON files and point them at your Prometheus-compatible
datasource.

| Dashboard              | Use it for                                                      |
| ---------------------- | ---------------------------------------------------------------- |
| **Supra / Overview**   | Triage entry point. Start here; panels link to the deep dives.  |
| **Supra / Consensus**  | Block commits, round duration, votes, certificates, peer liveness |
| **Supra / Execution**  | Per-VM execution latency and stage breakdown                    |
| **Supra / Mempool**    | Backlog depth and transaction admission                         |
| **Supra / Networking** | Peer connections and the shared p2p instruction queue           |
| **Supra / RPC**        | Request latency and dropped messages (RPC nodes)                |
| **Supra / Node Health**| Process RSS/CPU/fds and RocksDB memory, stalls, compaction debt |
| **Supra / DKG**        | Epoch-boundary key generation — see [dkg.md](./dkg.md)          |

Each has a `datasource` variable and an `instance` filter, so one Grafana can
serve every node you run.

## Annotations

Two annotations mark operational events on the timeseries panels where the event
plausibly explains a change in the plotted signal. Toggle them at the top of each
dashboard.

| Annotation       | Colour | Marks                                    | Scope                                     |
| ---------------- | ------ | ---------------------------------------- | ----------------------------------------- |
| **Epoch change** | purple | Each epoch transition                    | Cluster-wide — one marker per transition  |
| **Node restart** | red    | A node process starting                  | Per instance; hover text names the node   |

Both are driven by metrics that *are* the event's own timestamp, so the marker
lands at the event itself rather than at the scrape that noticed it. The restart
annotation also marks a node's initial start, and both respect the dashboard's
`instance` filter.

Node Health carries no epoch annotation deliberately — process metrics are
epoch-agnostic. To show an annotation on a panel that does not have it, add that
panel's id to the annotation's `filter.ids` in the dashboard JSON.

One limitation worth knowing: when the query step is coarser than the event
cadence — a multi-day range over minute-length epochs — only the latest event in
each step is marked.

## Peer liveness: are *other* operators' nodes voting?

Almost every consensus metric describes the local node. `supra_moonshot_recent_voters_current`
is the exception, and it is the one metric that lets you detect a problem on a
node you do not run.

It is a gauge, emitted by validators only, labelled by `vote_type`
(`prepare_normal`, `prepare_fallback`, `prepare_optimistic`, `commit`). Its value
is the number of **distinct peers** whose votes of that type your node received
in the last reporting interval (~10s). Find it on *Supra / Consensus* →
*Votes, proposals & certificates* → **"Unique recent voters by type"**.

How to read it:

- In a healthy N-validator network, `prepare_normal`, `commit`, and
  `prepare_optimistic` all sit near N. A persistent drop of one across
  `prepare_normal` and `commit`, seen network-wide, means a peer has stopped
  voting.
- `prepare_optimistic` well below N under sustained load means the optimistic
  path is not working as expected, even though the chain may look fine.
- `prepare_fallback` is normally **0** — it only carries votes during view
  changes. A non-zero line means the network is falling back to that path.
- Every vote type reports each interval, so an inactive path shows a flat `0`
  rather than the series vanishing. The line visibly lifts off zero when the path
  activates, instead of just going stale.

**Set the `instance` filter deliberately.** On *All*, the panel's `max` gives you
the network-wide count, which drops when any validator goes down. Narrowed to a
single node, it shows only that node's view — which is what catches a partition
isolating one operator while the rest of the network carries on unaffected.

The gauge tells you *how many* peers voted, not *which*. To get identities, run
the node at `debug` level: each interval the consensus aggregator logs the voter
identities it saw per vote type. Comparing that against the committee names the
peers that are silent.

> Why the count is windowed rather than per-block: a peer counts as recently
> active if it voted at all during the interval, whichever block it voted on and
> whether or not the certificate had already formed. A per-block tally would miss
> slow votes that arrive after their certificate.
