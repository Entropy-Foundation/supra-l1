# Supra Framework Change Log

Changes to the Supra Move framework and gas schedule since `aptosvm-v1.16_supra-v1.8.13`, the
framework tag released as v11.4.3.

This describes the changes in framework tags `aptosvm-v1.16_supra-v1.8.14` and
`aptosvm-v1.16_supra-v1.8.15`, the two tags in the window. Everything below is `_v1.8.14` except the
one entry attributed to `_v1.8.15`.

> **The node binary must reach every node — validators and RPC nodes alike — before this framework
> is published, and adoption must be confirmed rather than assumed.** There is no on-chain gate — the
> publish itself is the switch. An un-upgraded validator degrades silently; an un-upgraded RPC node
> fail-stops and needs a snapshot restore. See [Rollout](#rollout) at the end.

---

## New Feature Flags

None. `features.move` is byte-identical to `aptosvm-v1.16_supra-v1.8.13`, and no identifier was
added to the Rust `FeatureFlag` enum. None of the changes below requires a feature-flag governance
action; every one lands with the framework upgrade itself.

---

## Gas Schedule Changes

None. No gas parameter was added, renamed or revalued, and `LATEST_GAS_FEATURE_VERSION` is unchanged
at `RELEASE_V1_16_SUPRA_V1_8_0` (version 25).

---

## Modified Modules

### A DKG session no longer stores its members' keys

The change that motivates the release. A DKG session used to embed a full copy of every member's
`ValidatorPublicKeys` blob in every one of its committees — the dealer committee, each receiver
committee, and again in the start event. At 83 validators a session reached 1,017,168 bytes, 97% of
the `txn.max_bytes_per_write_op` gas limit of 1 MiB, and would have exceeded it at 86. The session is
written from the block prologue, so crossing that limit halts the chain.

A session now records only which validators comprise each committee. Their keys come from the
validator set, which is the single record of them. The same session at 83 validators is 24,449 bytes.

### `supra_framework::supra_dkg`

**New resources: `DkgSessionInProgress` and `DkgSessionLastCompleted`**

Supersede the two `Option` fields of `DKGState`, which is retained but no longer written after the
first session starts under the new layout. Splitting them gives each session its own write-op size
budget, so starting one no longer rewrites the other.

Both are created lazily, by the first `start_v2`, via `create_signer` — sessions are driven from the
block prologue and from VM-dispatched validator transactions, neither of which carries a framework
signer.

**New event: `DKGStartEventV2`**

Carries `dealer_epoch` and `start_time_us` only. `DKGStartEvent` embedded a third copy of the
session's committees; validators only ever read the epoch and the start time from a start event and
take the committees from the session resource. `DKGStartEvent` is retained but no longer emitted —
indexers and off-chain consumers that key on it must move to `DKGStartEventV2`.

**New function: `start_v2`**

Supersedes `start`, taking committees that record only their members. `start` is retained, and
nothing calls it: removing a published function is not an upgrade-compatible change.

**New view functions**

`incomplete_session_v2`, `last_completed_session_v2`, and `incomplete_session_dealer_epoch`. The last
returns just the epoch, for the block prologue's epoch-timeout check, which runs on every block for
as long as a session is open; `incomplete_session` copies the whole session out of storage.

**New committee types**

`DkgCommitteeV2` and `ReceiverCommitteeV2`, holding `vector<address>` rather than per-member key
blobs. Their constructors have `friend` visibility.

**`initialize` also creates the new session resources**

Except for the running-session slot while a session that predates it is still in progress: the VM's
validator for DKG transactions treats that slot's mere existence as meaning the new layout is
authoritative, so creating it mid-session would make it reject that session's own `set_dkg_meta` and
`finish`. `start_v2` creates the slot and retires the legacy session in one transaction, so the two
cannot disagree when it is left to do the work. This matters only if `initialize` is run from a
governance script, which it is documented as supporting; there is no ordering constraint on the
caller.

### `supra_framework::stake`

**New resource: `PreviousValidatorSet`**

The validator set as it stood during the epoch immediately preceding the current one, written by
`on_new_epoch` immediately before the epoch is incremented. `ValidatorSet` is replaced wholesale at
every epoch boundary, so a completed DKG session — dealt by one epoch's validators for the next
epoch's — could not otherwise resolve the validators that dealt it. There is no epoch field: the
resource is structurally always exactly one epoch behind.

Created lazily via `create_signer`, on the first `on_new_epoch` after this framework is published.

**New function: `set_dkg_output_keys_for_members`**

Installs DKG output keys against the membership the session pinned when it started, rather than
against a validator set recomputed at `finish`. The recomputation reads each validator's live
`ValidatorConfig`, where the session pinned the values in force when it started, so a validator that
rotated its consensus key in between made the two disagree.

**Breaking change.** A validator's next-epoch voting power is now its stake alone

`ValidatorInfo.voting_power` in `ValidatorSet` used to include the validator's pending staking
rewards and its share of collected transaction fees. It is now `active + pending_active +
pending_inactive`, counting `pending_inactive` only while its lockup has not expired. Rewards and
fees still accrue and are still paid into the pool at the epoch boundary; they simply count from the
epoch *after* the one in which they are earned, once they are part of `active`.

The reason is determinism across a DKG window. The DKG receiver committee is previewed when a session
starts and the same set is committed at the epoch boundary, and the two must be identical: the
committee order is the index basis the session's output key shares are matched to validators by. Stake
movements, joins, leaves and key rotations are all frozen for the duration of a reconfiguration, but
`block_prologue` advances validator performance statistics and the collected-fees table on every
block with no such guard, so rewards and fees were the only inputs that could move between the two
reads. Taking them out of the computation removes the last way the preview and the commit could
disagree.

Note that this figure has never governed governance voting: `supra_governance` is council-based, one
member one vote, which 1.8.14 records explicitly in the doc comments on `supra_vote` and
`supra_vote_internal`. Stake and `ValidatorSet` voting power play no part in it.

**Breaking change.** Next-set eligibility is judged on stake alone

Because the minimum-stake comparison uses the figure above, pending rewards can no longer carry a
validator over `minimum_stake`. A validator whose staked balance is below the minimum but within one
epoch's rewards of it is retained today and will be dropped at the first epoch boundary after this
framework is published. It can re-join once its stake alone clears the minimum.

Networks running `min_stake = 0`, which is the default (see the `[move_vm]` table in the node
configuration guide), are unaffected — every validator clears a minimum of zero.

**Breaking change.** `join_validator_set` rejects two cases it used to accept

Both were previously accepted at join time and then silently dropped at every epoch boundary, leaving
a validator that appeared to have joined but never activated and produced no error explaining why.
They now fail at the door with an actionable abort:

| Abort | Condition |
|---|---|
| `ESTAKE_TOO_LOW` (`0x10002`) | The pool clears `minimum_stake` only by counting `pending_inactive` stake whose lockup has expired. The admission gate now applies the same stake-only rule the epoch boundary will. The *maximum*-stake bound and the joining-power budget still count the whole pool, conservatively. |
| `EUNROTATED_LEGACY_CONSENSUS_KEY` (`0x3001a`) | The validator's stored consensus key is still a legacy bare-ed25519 key while the v2 identity format is enforced. Such a key cannot take part in DKG and was already excluded from every validator set. Rotate the consensus key first. |

The second applies only to pools carried over from before the v2 format change — registration already
rejects legacy key submissions under v2, and genesis is unaffected because the key format and the
feature flag are both derived from the same genesis feature set.

### `supra_framework::staking_config`

**Breaking change.** Staged staking-config updates now apply at the epoch boundary

`update_required_stake`, `update_recurring_lockup_duration_secs`, the reward-rate updaters and
`update_voting_power_increase_limit` took effect the moment the governance transaction executed. They
are now staged in `config_buffer` and applied at the end of the epoch transition, so a governance
change to `min_stake` — the one anticipated in the node configuration guide — becomes visible at the
start of the next epoch rather than immediately.

This is required by the same determinism property as above: the epoch transition itself reads the
staking config, so a value that changed part-way through an epoch would be read differently by the
DKG preview and by the boundary that seals the set the session was built for. Staging it means both
reads see the values the epoch started with.

**Note for governance scripts:** a proposal that changes a staking parameter and then depends on the
new value in the same transaction will read the old one. Split it across two proposals.

### `supra_framework::reconfiguration`

`reconfigure()` now calls `staking_config::on_new_epoch()`, after `stake::on_new_epoch()`. This is
where the staged staking update above is applied. It sits here rather than in
`reconfiguration_with_dkg::finish` because `reconfigure()` is the one point every epoch-transition
path funnels through — including `block_prologue`'s epoch timeout, which is the live path while DKG
is disabled, and the immediate setters in `gas_schedule`, `version`, `execution_config` and
`consensus_config`. One consequence is deliberate: any action that advances the epoch, including an
otherwise unrelated config setter, applies a staged staking update.

### `supra_framework::validator_public_keys`

No signature change. The module is now read as the single record of a validator's keys, since
sessions no longer carry copies of them.

### `supra_framework::config_buffer` and `supra_framework::create_signer`

`friend` declarations only, enabling the above: `config_buffer` befriends `staking_config`, and
`create_signer` befriends `supra_dkg` and `stake` so each can create its new resources lazily rather
than requiring a genesis migration.

### `supra_framework::supra_governance`

Documentation only. The doc comments on `supra_vote` and `supra_vote_internal` now state that voting
is council-based — the voter must be a member of `SupraGovernanceConfig.voters` and each member
carries one vote — rather than describing the stake-pool-weighted voting inherited from upstream,
which Supra does not use.

---

## Native Function Changes

None. No Move signature and no native implementation changed.

---

## VM Changes

Shipped by the same tag, and the main reason the binary must lead the framework: transaction
validation for DKG validator transactions now resolves the running session from whichever layout
holds it. `DkgSessionInProgress` is authoritative once it exists; until then a session started before
it existed may still be running in the superseded `DKGState` slot. Every read distinguishes an absent
resource from one it could not read — treating a read failure as absence would route a running
session to the empty slot and report `DKG_SESSION_NOT_IN_PROGRESS` while a session runs, discarding
every `set_dkg_meta` and `finish` for it.

**This is a change to execution semantics, not to validation alone.** `AptosVM` executes a DKG
validator transaction by first validating it and turning a rejection into
`TransactionStatus::Discard`, so the resolver above runs inside execution, on every node, as part of
producing the block's result. Two binaries that resolve the session differently therefore produce
different transaction outcomes for the same block — which is a chain split, not a degraded service.
Consequences are under [Rollout](#rollout).

---

## Rollout

The node release must reach **every node** — validators and RPC nodes alike — **before** this
framework is published, and adoption should be confirmed rather than assumed. There is no on-chain
gate: the publish itself is the switch.

### An un-upgraded RPC node fail-stops

The most severe consequence, and the one least likely to be anticipated. `start_v2` clears
`DKGState.in_progress` as it writes `DkgSessionInProgress`, so a pre-v11.5.0 VM finds no running
session in the only slot it knows about and **discards** the session's `set_dkg_meta` and
`finish_with_dkg_result` with `DKG_SESSION_NOT_IN_PROGRESS`, where an upgraded node keeps and
executes them. The two nodes compute different Move transaction-inclusion Merkle roots for that
block, the un-upgraded node's root does not match the one the committee signed, and it terminates
rather than serve a view it knows is diverged:

```
ERROR task_manager::shutdown: Data inconsistency detected: received certificate for block height …
with valid committee signatures but with invalid data which states that this this node is
inconsistent with the majority view. Please upgrade to the latest binary and restore from the
latest snapshot.
```

This fires at the first epoch boundary after the framework is published — the first block carrying
DKG transactions written under the new layout. Recovery is to upgrade the binary **and** restore from
a snapshot: the node's state diverged at that block, so restarting on the old binary only reproduces
the halt, and restarting on the new one resumes from diverged state. This was confirmed on testnet by
deliberately holding two RPC nodes on `supra_rpc_v11.4.3` through the framework upgrade; both halted
as described.

### An un-upgraded validator degrades silently

Two independent reasons, both of which stop it from taking part in the DKG:

- It cannot read `DkgSessionInProgress` or `DkgSessionLastCompleted`, so it sees no running session.
- `start_v2` emits only `DKGStartEventV2`, whose type tag an older binary does not recognize, so the
  event is ignored rather than acted on.

Neither is loud. An un-upgraded validator does not error; it silently stops participating. If enough
of them are still on the old binary for the session to miss its threshold, no keys are produced for
the next epoch and the chain stops advancing epochs. And once the first session completes under the
new layout, `finish` empties the superseded `DKGState.last_completed`, at which point an old binary
panics at every epoch boundary rather than degrading. A validator is also subject to the fail-stop
above, for the same reason an RPC node is — it executes the same blocks.

### A session already running when the framework is published

It completes through the pre-upgrade path; `set_dkg_meta` and `finish` keep driving it through the
superseded resource, because its committees only exist there. Nothing above applies to it — the
layouts disagree only from the first `start_v2` onwards. The next session to start uses the new
layout, and the session after that is the first for which both slots live there.

Note that this was relevant for testnet, but is not for mainnet because DKG is not active there yet.
