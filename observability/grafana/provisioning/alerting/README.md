# Supra alert rules

These files provision Grafana **alert rules**, a **contact point**, and a
**notification policy** for the Supra node dashboards. They load automatically:
`compose.yml` mounts the `provisioning/` directory into Grafana at
`/etc/grafana/provisioning`, and Grafana applies changes on restart.

| File                 | Purpose                                                       |
| -------------------- | ------------------------------------------------------------- |
| `rules.yaml`         | The SLO alert rules (group `supra-slo`)                       |
| `contactpoints.yaml` | Slack contact point `supra-slack` (**placeholder** webhook)   |
| `policies.yaml`      | Routes alerts labelled `team=supra-node` to `supra-slack`     |

All rules query the **Mimir** datasource (`uid: mimir`) and appear in a Grafana
folder called **Supra Alerts**, under *Alerting → Alert rules*. Verify
provisioning with:

```
curl -u admin:admin localhost:3000/api/v1/provisioning/alert-rules
```

> **These rules are a starting point, not a finished alerting policy.** The
> thresholds marked `TODO(baseline)` are conservative placeholders chosen without
> knowledge of your hardware, network size, or load. Read the tuning section
> below and set them from your own observed baselines before you page anyone on
> them.

## The `supra-slo` rules

| Alert                        | Metric                                            | Condition (default)                             | `for` | Severity |
| ---------------------------- | ------------------------------------------------- | ----------------------------------------------- | ----- | -------- |
| Epoch not advancing          | `supra_lifecycle_epoch_current`                   | `changes(...[125m]) == 0` (or NoData)           | 5m    | critical |
| Commit latency p95           | `supra_blocks_commit_latency_seconds_bucket`      | p95 > 5s — `TODO(baseline)`                     | 10m   | warning  |
| Move execution latency p95   | `supra_execution_move_block_execution_seconds_bucket` | p95 > 2s — `TODO(baseline)`                 | 10m   | warning  |
| EVM execution latency p95    | `supra_execution_evm_block_execution_seconds_bucket`  | p95 > 2s — `TODO(baseline)`, NoData = OK    | 10m   | warning  |
| TPS below floor              | `supra_transactions_executed_total` + backlog     | rate < 1 tx/s while backlog > 0 — `TODO(baseline)` | 15m | warning |
| Mempool backlog growing      | `supra_mempool_backlog_transactions_current`      | `deriv(...[15m]) > 0`                           | 30m   | warning  |
| RocksDB write stall          | `supra_rocksdb_write_stall_active_current`        | `> 0` per `db`                                  | 5m    | critical |
| RocksDB compaction debt      | `supra_rocksdb_pending_compaction_bytes_current`  | sum per `db` > 10 GiB — `TODO(baseline)`        | 15m   | warning  |
| Process RSS growth           | `process_resident_memory_bytes`                   | `delta(...[6h]) > 2 GiB` — `TODO(baseline)`     | 30m   | warning  |
| RPC messages dropped         | `supra_rpc_messages_dropped_total`                | `rate(...[5m]) > 0`, NoData = OK                | 5m    | warning  |

The consensus and storage rules alert on validators; the RPC drop rule alerts on
RPC nodes.

### Two `noDataState: OK` settings you must not "fix"

Both look like oversights and are not:

- **EVM execution latency** queries a series that only exists on builds with EVM
  support that are actually executing EVM transactions. A deployment without it
  must not page.
- **RPC messages dropped** queries a counter, and counters are absent from the
  Prometheus exposition until first incremented. A *healthy* node therefore has
  no series at all — with any other `noDataState`, this rule fires precisely when
  nothing is wrong.

## Tuning the thresholds

> Grafana does **not** interpolate environment variables in unified-alerting
> provisioning files — a `${VAR}` is stored verbatim and the query then fails
> with `bad_data: invalid parameter`. Env-var interpolation works only for
> datasource and dashboard providers. Every tunable value below is therefore
> edited directly in the YAML, followed by a Grafana restart.

- **Epoch not advancing** — the lookback window is the literal `[125m]` in the
  rule's `expr`, repeated in its description. It **must exceed your longest
  expected epoch**, or it fires during normal operation. Edit both occurrences.
  `noDataState: Alerting` here is deliberate: a node that stops being scraped is
  itself worth paging on.
- **Commit and execution latency** — the `params: [5]` (commit) and `params: [2]`
  (per-VM execution) evaluators are in seconds. Set each to a comfortable
  multiple of your observed p95, which you can read off the *Supra / Consensus*
  and *Supra / Execution* dashboards. All latency histograms share buckets
  spanning 1 ms to 30 s, and `histogram_quantile` saturates at the top bucket, so
  keep thresholds well below 30 s or the rule stops discriminating.
- **TPS below floor** — `params: [1]` is in tx/s. Set it well below your observed
  normal throughput. The backlog qualifier means an idle chain never fires; keep
  `noDataState: OK` for the same reason.
- **Mempool backlog growing** — fires on a sustained positive slope held for 30m.
  If it is noisy on a healthy network, raise the evaluator above `0` (to a few
  tx/s), lengthen `for`, or switch to the lower-noise counter form noted in
  `rules.yaml`.
- **RocksDB compaction debt** — one global 10 GiB threshold. Per-`db` thresholds
  are worth setting once you have baselines, since the chain and archive
  databases have very different write profiles.
- **Process RSS growth** — `delta()` over 6h, because RSS is a gauge and
  `increase()` is counters-only. Raise the byte threshold if legitimate cache
  warm-up trips it.

## Wiring up notifications

`contactpoints.yaml` ships with a placeholder webhook so that provisioning loads
cleanly. Alerts **evaluate but are not delivered** until you replace it. As
above, an environment variable will not work — the URL must be literal.

1. Create an [Incoming Webhook](https://api.slack.com/messaging/webhooks) in
   Slack, or configure a different contact point type entirely.
2. Replace the `url:` value in `contactpoints.yaml`, or set it in the Grafana UI
   under *Alerting → Contact points → supra-slack*.
3. Recreate Grafana so the contact point is re-provisioned:

   ```
   docker compose -f observability/compose.yml up -d --force-recreate grafana
   ```

4. Verify under *Alerting → Contact points → supra-slack → Test*.
