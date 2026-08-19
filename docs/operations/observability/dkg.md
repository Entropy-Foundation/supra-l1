# DKG observability

How to tell, from metrics and logs alone, whether a Distributed Key Generation round is
progressing, slow, or wedged — and whether a validator has silently stopped doing work while
waiting for its threshold keys.

Everything below is read from the released validator binary's metrics endpoint and its logs; no
access to the node source is assumed.

## Why this matters

DKG runs once per epoch boundary and the epoch transition depends on its output. A validator
that is in the next committee, ran the DKG, and never received its BLS threshold key shares
holds its transition indefinitely — and by that point every consensus component is already
paused by the ack-gated `EndEpoch` fanout, so the node sits silent. The metrics below exist so
that state is visible instead of indistinguishable from a healthy node.

The **Supra / DKG** Grafana dashboard (`observability/dashboards/json/supra-dkg.json`) is built
around exactly these series.

## The phase gauge

`supra.dkg.phase.current` is the first thing to look at. It is a small integer:

| Value | Phase          | What the node is doing |
| ----- | -------------- | ---------------------- |
| 0     | `Idle`         | No session is running. |
| 1     | `Dealing`      | Distributing dealings, collecting signatures on them, voting on the dealing meta and the DKG meta — up until the `DKGMetaQC` is observed on chain. |
| 2     | `Aggregation`  | Fanning out commitments over Deliver / CS-Deliver and aggregating dealings into threshold key shares. |
| 3     | `Done`         | This node derived its own output; it now only serves recovery requests. |
| 4     | `HoldingKeys`  | The session finished and the node is holding its output keys until `EndEpoch` arrives. |

The values are the metric's contract with dashboards and alerts; they are stable across releases.

Phases 1 and 2 each have a duration histogram (`dkg.dealing_phase_duration.seconds`,
`dkg.aggregation_phase_duration.seconds`), plus `dkg.session_duration.seconds` for the whole
round. A session recovered by syncing rather than started from a `DKGStartEvent` also begins in
`Dealing` and may leave immediately, so a very short dealing phase indicates a restart rather
than a fast round — cross-check `dkg.timeout_restarts.total` and `dkg.sessions_started.total`.

## Progress against thresholds

A raw count says nothing without its threshold, so each progress gauge is paired with the
number it has to reach. These four carry a `committee_index` label, one series per receiver
committee, because a round can be short on one committee while fine on another:

| Collected | Required | Phase |
| --------- | -------- | ----- |
| `dkg.dealing_signatures_collected.current` | `dkg.dealing_signatures_required.current` | Dealing |
| `dkg.commitments_received.current` | `dkg.commitments_expected.current` | Aggregation |
| `dkg.receiver_completions_received.current` | `dkg.receiver_completions_required.current` | Aggregation |

`commitments_expected` is 0 until the DKG meta commitments arrive, so a flat `0/0` means the node is
stuck *before* the Deliver phase rather than part-way through it.

> The `committee_index` label values are pre-rendered and bounded: indices past a fixed limit
> collapse into a single `overflow` series. Seeing `overflow` is not something you can fix
> locally — report it to Supra, as the bound needs to be raised in the node.

All six of these gauges are zeroed when the runner returns to `Idle`, for every `committee_index`
value they can carry. The Prometheus exporter runs without an idle timeout, so a series that
simply stopped being written would keep reporting its last value until the process restarted —
and because the two aggregation pairs are written *only* during aggregation, the next session's
dealing phase would otherwise still show the previous session's completed counts. Reading them
between sessions therefore gives 0, not the last round's result: use
`dkg.session_duration.seconds` for what the previous round did.

## Message volume

Counters for each protocol message type: `dealings_sent`/`_received`,
`dealing_signatures_received`, `encrypted_dealings_sent`/`_received`,
`dealing_meta_votes_received`, `dkg_meta_votes_received`, `public_shares_votes_received`, and
`deliver_codewords_sent`/`_received` plus the `cs_deliver_` equivalents.

Two caveats:

- `deliver_codewords_sent` counts **one per emitted message**, so a Deliver broadcast to the
  clan counts once, not once per peer. `cs_deliver_codewords_sent` is per recipient.
- The `_received` code-word counters include messages for a stale epoch, which are counted and
  then dropped.

### Dropped messages

Two counters, one per direction, and either can stall a round on its own because both are
silent from the other end's point of view and nothing resends automatically:

- `dkg.inbound_messages_dropped.total` — a message this node failed to take delivery of, because
  the runner's channel was full.
- `dkg.outbound_messages_dropped.total` — a message this node failed to send, because the p2p
  instruction channel was still full after every retry (`reason="full"`) or the network task had
  shut down (`reason="closed"`). `dkg.outbound_send_retries.total` is the leading indicator:
  a non-zero rate there means the channel is saturating under DKG load but sends are still
  getting through.

The outbound pair is not independent of `network.send_errors.total`. Every failed attempt also
passes through the metered network client, so one saturation event raises that counter by
roughly the retries plus the drops. Read them as two views of one incident, not two problems.
The `reason` label values are shared (`full` / `closed`), so the series compare directly.

All three are lagging indicators: they only move once messages are already being lost. The
leading one is `network.instruction_queue_depth.current` against
`network.instruction_queue_capacity.current` — the shared p2p instruction channel filling up.
See *Instruction queue saturation* on the **Supra / Networking** dashboard.
Watch the ratio during a DKG round: it climbing towards 1 is the warning that the next burst
will start dropping. Note that the channel is shared with consensus, mempool, the epoch manager
and the commitment signer, so a rising depth is not necessarily the DKG's doing; compare against
the per-service inbound counters to see whose traffic is behind it.

## Reading a stall

**Is it slow or is it wedged?**

1. Look at `dkg.phase.current` across validators. If they are spread across phases and moving,
   it is slow, not wedged. If one validator sits in a phase the others have left, that node is
   the problem.
2. Compare the collected/required pair for that phase. A collected line that flattens below its
   required line names the missing input — e.g. a dealer whose dealing too few receivers signed.
3. Check `dkg.timeout_restarts.total`. A growing gap between `dkg.sessions_started.total` and
   `dkg.sessions_completed.total` is a round that keeps restarting without producing output.
4. Compare `dkg.session_duration.seconds` p95 against the configured `dkg_timeout_ms`
   (see [config.md](../node-configuration/supra/config.md)). A p95 approaching the timeout
   means restarts are imminent.

**Is a validator paused waiting for keys?**

`dkg.waiting_for_threshold_keys_epoch.current` is the epoch a validator is holding its
transition for, and 0 when it is not waiting. Non-zero means consensus on that node is paused.
The node logs the hold at `INFO` when the gate is first hit, then re-reports it every
30 seconds, escalating to `WARN` once the transition has been held longer than 30 seconds — so
the condition is visible as an ongoing state rather than a single line that scrolls away.

**Alert on it with a `for:` duration, not on `!= 0` alone.** Reaching this gate is ordinary:
a validator commonly arrives before its shares do and clears within a second or two — under two
seconds on a 7-node local network. The gauge is a state signal and is set for the whole of that
window, so a bare `!= 0` rule fires on healthy nodes whenever a scrape lands inside it. Require
the condition to hold for a minute or two, comfortably above a normal wait and far below the DKG
timeout:

```yaml
- alert: SupraDkgValidatorPausedOnThresholdKeys
  expr: supra_dkg_waiting_for_threshold_keys_epoch_current != 0
  for: 2m
```

The same reasoning is why the log line only escalates to `WARN` after 30 seconds: a warning that
fires on the healthy path is one operators learn to ignore.

A validator here is waiting on the DKG output of *others* as much as its own: it is in the next
committee, the DKG ran, and its shares never arrived. Correlate with the phase gauge on the rest
of the committee to find out whether the round completed at all.

## Log lines

Production log levels do not show `debug!`, so the milestones are at `info!`:

- session start, with the roles this node plays and every committee size and threshold — the
  anchor for reading every later progress line
- the dealing-signature threshold being crossed, and the dealing-signature timer expiring
  (at `warn!` if the threshold was not met, because that dealer now holds up the round)
- all Deliver commitments received; dealings aggregated
- each threshold key share derived, per receiver committee
- the DKGMetaQC being observed, with time spent in the dealing phase
- every phase transition, with its elapsed duration
- the public-shares QC and the DKGMeta transaction submissions

Per-message lines stay at `debug!` on purpose: at mainnet committee sizes the encrypted-dealing
and commitment traffic is O(n²), which would swamp the log. Use the counters and the paired
progress gauges for those instead.
