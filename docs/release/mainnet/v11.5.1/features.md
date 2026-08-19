# Feature Activation

The four feature flags shipped with `supra_node_v11.5.1`, what each one changes, and the order they
must be enabled in.

The other changelogs in this directory describe what the binary and the framework do differently in
this release. This one describes what the network gains when governance turns the four feature flags
on. That is always a separate step from the release: the release ships the code, and a governance
proposal flips the flags afterwards.

> **On Supra Mainnet all four flags are active** — they took effect at epoch 7575 on 2026-08-14.
> See [the mainnet status page](../README.md) for the dates and for what the activation means if
> you run a validator. This document describes the flags themselves and applies to any network;
> whether a given network has enabled them is recorded per network.

| Flag | Id | Lifetime | Who it affects |
|---|---|---|---|
| `SUPRA_BLS_KEYS` | 97 | permanent | Node operators (key format), integrators (validator-set parsing) |
| `SUPRA_BCFT_CERTIFICATES` | 98 | permanent | Integrators verifying certificates off-chain |
| `SUPRA_DKG` | 99 | transient | Node operators, smart-contract developers |
| `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` | 100 | permanent | Integrators, light clients, bridges |

**The order matters and is not enforced on chain.** `SUPRA_BLS_KEYS` must be on before `SUPRA_DKG`,
because the v2 validator identity is what holds the key material the DKG deals against. The default
DKG configuration additionally requires `SUPRA_BCFT_CERTIFICATES`, because the receiver committees it
asks for are defined in terms of the BCFT threshold types. Activating them together, in the order
above, satisfies both constraints.

## When activation takes effect

The flags are buffered with `features::change_feature_flags_for_next_epoch`, so they apply at the
**next epoch boundary**, not when the proposal executes.

`SUPRA_DKG` has one further step of delay that is worth understanding, because it decides which epoch
boundary is the first one a DKG governs. Each validator reads the flag per block to choose which
block prologue to run, so the boundary that *applies* the buffered flags is itself still executed by
the ordinary prologue. The extended prologue — and with it the reconfiguration path that starts a DKG
— applies from the following block onwards. The first epoch change produced by a DKG is therefore the
**second** boundary after the proposal executes.

---

## Prerequisites

Two things must be in place before the flags are enabled. On Supra Mainnet both are performed by a
preceding governance action, and neither is something an operator does.

**The `randomness::PerBlockRandomness` and `reconfiguration_state::State` resources must be
published.** The extended block prologue that `SUPRA_DKG` selects reads both. Mainnet genesis was
produced by the mainnet genesis encoder, which does not publish either, so they are published by
governance instead. Both initializers are idempotent.

**The DKG configuration must be set.** `dkg_config::set_for_next_epoch` installs it, and the default
configuration is a quorum dealer committee plus three receiver committees over the same membership —
the next epoch's validators — differing only in threshold type:

| Receiver committee | DKG threshold type | Resharing |
|---|---|---|
| 1 | `BcftQuorum` | no |
| 2 | `BcftValidity` | no |
| 3 | `ClanMajority` | no |

Each committee produces one set of threshold key shares, which is why a validator's on-chain identity
carries one BLS threshold key per threshold type rather than a single one. The three committees are
what let a certificate be signed at the threshold appropriate to what it certifies: `BcftQuorum` for
block quorum certificates and committee authorizations, `BcftValidity` for batch availability, and
`ClanMajority` for transaction-inclusion certificates.

---

## `SUPRA_BLS_KEYS` — validator identity v2 (97)

### What changes

A validator's on-chain consensus key stops being a bare 32-byte Ed25519 public key and becomes a BCS
`validator_public_keys::ValidatorPublicKeys` blob holding the full set:

- the network key,
- an aggregatable BLS multisignature key,
- a class-group key, used by the DKG,
- the Ed25519 key,
- and, once a DKG has run, one BLS **threshold** key share per certificate threshold type.

Consensus messages change signature scheme to match. Prepare votes, batch votes and certifier votes
move from Ed25519 to BLS multisignatures, and committees are constructed as `CommitteeV1` rather than
`CommitteeV0`. A validator committee is only ever built whole: if the on-chain record for any member
cannot be resolved, no committee is formed for that epoch rather than a short one. This is required
because the DKG deals its output shares against a committee's member ordering, so a committee that
differed between two validators would match shares to the wrong members.

### What node operators must do

**Rotate to the v2 key format.** Where the flag is not yet enabled, do it before it is. Where it is
already enabled — as it is on Mainnet — a validator whose stored consensus key is still a bare
Ed25519 key is not eligible for the active validator set, with the same effect as falling below the
minimum stake: it stays registered but stops being included in the validator set at each epoch
boundary until it rotates. `supra node identity rotate-keys` performs the rotation. Since v11.5.1
that command also migrates a pre-v11 `node_identity.pem` automatically; see the
[Supra CLI change log](./supra_cli.md).

A validator that was dropped for this reason recovers by rotating and then calling
`join_validator_set`; its stake is untouched throughout. The
[mainnet status page](../README.md) sets out that recovery path.

`join_validator_set` also rejects an unrotated legacy key outright, with
`EUNROTATED_LEGACY_CONSENSUS_KEY` (`0x3001a`), rather than accepting the join and then never
activating the validator. See the [Supra framework change log](./supra_framework.md).

After the flag is on, a rotation **merges** only the static keys into the stored record and leaves the
DKG-written threshold key shares in place, so rotating a key does not cost a validator its shares or
force it to wait for the next DKG. Rotation submissions must carry a BLS multisignature
proof-of-possession over the new keys; the CLI produces it.

### What integrators must do

Anything that reads `ValidatorConfig.consensus_pubkey` — validator-set explorers, staking dashboards,
monitoring — must parse it as a BCS `ValidatorPublicKeys` blob instead of raw key bytes. The v4
consensus endpoints expose the parsed form, so clients that read committees from the API rather than
from the resource need no change.

---

## `SUPRA_BCFT_CERTIFICATES` — BCFT certificate thresholds (98)

### What BCFT means

Byzantine *and Crash* Fault Tolerant. The classic BFT model budgets for `f` Byzantine members out of
`n = 3f + 1`. The BCFT model budgets separately for `f` Byzantine and `c` crash-only members, sized
for `n = 3f + 2c + 1 + k` (`0 ≤ k < 5`) with `c = f`, so `f = ⌊(n − 1) / 5⌋`. Tolerating crashes
explicitly, rather than counting a crashed member against the Byzantine budget, is what lets the
protocol keep its safety margin on a network where nodes go offline for ordinary reasons.

This flag does not introduce a new kind of certificate. It changes which threshold each existing
certificate is formed at:

| Certificate | Threshold before | Threshold after | Formula after |
|---|---|---|---|
| Block quorum certificate (`SmrQC`) | `Quorum` | `BcftQuorum` | `2f + c + 1` |
| Committee authorization | `Validity` | `BcftQuorum` | `2f + c + 1` |
| Batch availability certificate | `Validity` | `BcftValidity` | `f + 1` |
| Timeout certificate | `Quorum` | `BcftFallbackViewChange` | `n − f − c` |

Thresholds are evaluated against committee voting weight. For committees of five members or fewer the
BCFT validity and quorum thresholds fall back to their non-BCFT equivalents, so that the threshold
stays meaningful on a very small network.

The committee authorization is raised the furthest, from `f + 1` to `2f + c + 1`. It is the material
an external verifier uses to establish which keys are authorized for an epoch, and for a cross-chain
verifier it may be the only material it has, so it is deliberately formed at a security threshold
rather than the cheapest one that would carry consensus.

Transaction-inclusion certificates are unaffected: they are formed at `ClanMajority` (`f + 1` for
`n = 2f + 1`) both before and after.

### What this means for integrators

**The threshold type is part of what is signed.** It is fed into the certificate digest as a fixed
byte string, so the same set of votes produces a different digest before and after the flip. Two
consequences:

- An off-chain verifier must accept the BCFT threshold types. In JSON a certificate's `kind` field
  carries the variant name — `"BcftValidity"`, `"BcftQuorum"`, `"BcftFallbackViewChange"` — and in
  binary encodings the discriminants are `3`, `4` and `5`. The names and discriminants are stable.
- A client that caches certificate digests across the activating epoch boundary must re-fetch them.

Because the change is a coordinated one, every validator must be running a binary that knows the BCFT
threshold types before the flag is enabled. It takes effect at an epoch boundary, and validators do
not process consensus messages from a future epoch, so there is no window in which two thresholds are
in force at once.

---

## `SUPRA_DKG` — distributed key generation (99)

### What changes for the network

Validators run a distributed key generation round at each epoch boundary, and from the following
epoch each validator holds BLS **threshold** key shares for its committees. Three things follow.

**Commit votes become threshold signatures.** A commit quorum certificate carries a single aggregated
BLS threshold signature rather than a collection of individual ones. It is compact, deterministic —
the same committed block always yields the same signature regardless of which subset of validators
contributed — and verifiable against the committee's threshold public key by anyone, including an
on-chain verifier on another chain.

**Reconfiguration becomes asynchronous.** Before activation, `supra_governance::reconfigure` ended
the epoch within the transaction that called it. After activation it *starts* a DKG instead, and the
new epoch begins in a later block once the DKG completes. Any on-chain configuration change that
relies on a reconfiguration — including a feature-flag change — therefore lands a few blocks later
than it used to, rather than at the end of the executing transaction.

**Epoch-boundary blocks are no longer required to be empty.** Before activation, the block at an
epoch boundary and its same-epoch descendants had to carry no payload. That restriction is lifted,
because DKG validator transactions have to travel in exactly those blocks.

### What node operators must know

- **DKG is CPU-bound work that runs at every epoch boundary.** It is sized by
  `dkg_thread_pool_size`; see the [node configuration change log](./config.md) for how that
  field interacts with the node's other CPU consumers, and the
  [node configuration guide](../../../operations/node-configuration/supra/config.md) for sizing guidance.
- **A validator that does not receive its threshold key shares for the new epoch holds its own epoch
  transition** until they arrive. By that point its consensus components are already paused for the
  transition, so the node is quiet rather than obviously failing. This state is instrumented; watch
  `supra.dkg.phase.current` and the paired progress/threshold gauges described in
  [DKG observability](../../../operations/observability/dkg.md), and the **Supra / DKG** Grafana dashboard built
  around them.
- **Timeouts are generous by design.** `dkg_timeout_ms` defaults to 300 s in this release, and
  `dealing_signature_collection_timeout_ms` to 10 s. Both are documented in the
  [node configuration change log](./config.md), including which of the compiled default and
  the on-chain `SupraConfig` value is in force — each validator logs that on every epoch change.

### What smart-contract developers gain

On-chain randomness. See [Native unbiasable randomness](#native-unbiasable-randomness) below.

---

## `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` — transaction inclusion proofs (100)

### What is produced

Every block, each node accumulates the block's transactions into a per-VM Merkle accumulator — one
for MoveVM transactions and one for EVM transactions — and certifies the resulting roots.

The accumulator is a history tree, so the root at height *h* commits to every transaction up to
height *h*, and a proof for an old transaction stays valid as the chain grows. It is a two-level
structure:

- A **transaction leaf** is `keccak256(TX_DOMAIN ‖ tx_hash ‖ tx_events_root)`, binding each
  transaction to the root of its own events tree.
- An **event leaf** is the plain `keccak256` of the encoded event — a Move `ContractEvent`, or an EVM
  log as address, then topics, then data. Event leaves carry no domain separator; the internal nodes
  of the events tree carry `EVENT_DOMAIN`.

The two domain separators are the 32-byte hashes of their names,
`TX_DOMAIN = keccak256("SUPRA::TransactionAccumulator")` and
`EVENT_DOMAIN = keccak256("SUPRA::EventAccumulator")`. Hashing the name rather than prefixing it
directly keeps the separator fixed-length. Both trees use `keccak256` throughout so that a proof can
be verified by an EVM contract with no library beyond `keccak256`.

The trust anchor is the **transaction-inclusion certificate**: the validator committee's aggregate
signature over `{ movevm_merkle_accumulator_tree_root, evm_merkle_accumulator_tree_root, block_height,
epoch_id }`, formed at the `ClanMajority` threshold. A verifier that trusts the committee needs
nothing else — not the block, not the node that served the proof.

### What is exposed

None of these endpoints require authentication.

| Endpoint | Purpose |
|---|---|
| `GET /rpc/v4/transactions/certificates?start_height=&end_height=` | The inclusion certificates for a height range. `start_height` inclusive, `end_height` exclusive, both required, at most 100 heights returned. |
| `POST /rpc/v4/proofs/events` | Batch: proofs for many events across many transactions, all anchored to one certificate. At most 100 items and 500 resolved events. |
| `GET /rpc/v4/proofs/events/{event_hash}/transaction/{transaction_hash}` | A single event-emission proof. |
| `include_proof=true` on `GET /rpc/v4/transactions/{hash}`, `GET /rpc/v4/block/height/{height}`, `GET /rpc/v4/events/{event_type}` | Attaches `inclusion_proof` (or `proofs`, on events) to the ordinary response. |
| `GET /rpc/v4/consensus/committees/{epoch}` | The epoch's committee, including its `bls_threshold_public_keys` keyed by threshold type. Needed to verify a certificate. |

A proof carries `proof.siblings` (bottom-level first, toward the root), `leaf_index`,
`leaf_hash_value`, `merkle_root_hash_value` and `certified_at_height`. The batch response's `vm` field
is `"move"` or `"evm"` and tells the verifier which of the certificate's two roots to check against.

Note that proofs served by a node are anchored to a **certified** height, which trails the node's
latest executed block. The certificate in a batch response names the height the whole response is
anchored to; a proof cannot be produced for a transaction that has not yet been certified, and such a
request is answered `503` rather than with a proof that would not verify.

Inclusion data is pruned by epoch. A request for a height that has been pruned is answered `410 Gone`
with an `x-supra-oldest-block` header naming the oldest height still available, and within a page of
results a pruned entry is returned as `null` rather than failing the request.

### Verifying a proof end to end

1. **Fetch the certificate** for the height, from `GET /rpc/v4/transactions/certificates` (or take it
   from the batch response, which embeds it).
2. **Fetch the committee** for the certificate's `epoch_id.epoch`, from
   `GET /rpc/v4/consensus/committees/{epoch}`, and take the threshold public key for the
   certificate's `kind` — `ClanMajority` for an inclusion certificate.
3. **Verify the certificate's aggregate signature** over its `data` under that key. From here the two
   accumulator roots are trusted.
4. **Walk the event proof** to the events root: fold each sibling as
   `keccak256(EVENT_DOMAIN ‖ left ‖ right)`, taking the order of each pair from the leaf index.
5. **Walk the transaction proof** to the accumulator root the same way, with `TX_DOMAIN`, and check
   that it equals the root in the certificate — `movevm_merkle_accumulator_tree_root` for `"move"`,
   `evm_merkle_accumulator_tree_root` for `"evm"`.

Steps 1–3 are per height; steps 4–5 are per event. That asymmetry is what the batch endpoint exists
for: one signature verification amortized over many Merkle paths.

### What node operators should expect

Extra per-block work (two accumulator updates and one durable certification request), extra storage
for the accumulators and certificates, and a pruning job that runs at epoch boundaries. The one
operator-facing tunable is `[commitments] proposal_retry_delay_ms`, which sets how often a validator
retries certificate collection; see the [node configuration guide](../../../operations/node-configuration/supra/config.md).

The flag takes effect at an epoch boundary in both directions, and is safe to disable again at one.

---

## What These Features Enable

The four flags are infrastructure. What they are for is best seen through the two things that consume
them.

### SupraNova

SupraNova is Supra's cross-chain communication framework. Its HyperNova component bridges Ethereum
and Supra by verifying, on each chain, the other chain's own consensus signatures — rather than by
trusting a set of bridge operators. The token bridge is the first service built on it. See
[docs.supra.com/supranova](https://docs.supra.com/supranova).

For the Supra direction, the verifier is a set of Solidity contracts on the destination chain, and
what it verifies is exactly the material these flags produce:

| Verifier step | Flag it depends on |
|---|---|
| Read the epoch committee's BLS threshold public keys for `BcftQuorum` and `ClanMajority`, and check aggregate signatures with the BLS12-381 precompiles | `SUPRA_BLS_KEYS` (97) — the threshold keys exist only under the v2 identity, and only once a DKG has dealt them |
| Accept a committee authorization formed at `BcftQuorum` and an inclusion certificate formed at `ClanMajority` | `SUPRA_BCFT_CERTIFICATES` (98) — the threshold type is inside the signed digest, so the verifier must expect the BCFT variant |
| Walk the two-level Merkle path with the `SUPRA::TransactionAccumulator` and `SUPRA::EventAccumulator` domain separators to prove a Move event was emitted | `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` (100) — the proofs and the certificate over their roots |

So the bridge is built on flags 97, 98 and 100 together: 100 provides the proof that an event
happened, 98 fixes the threshold the proof's anchor is formed at, and 97 provides the keys that sign
it. It consumes the endpoints listed above, and one more —
`GET /rpc/v4/consensus/committee_authorization/{epoch}`, which is authenticated.

It does **not** depend on `SUPRA_DKG` for randomness. It depends on DKG only in that the threshold
keys it verifies against are what a DKG produces.

Note for anyone operating the off-chain side: because a verifier caches a committee per epoch, it
needs the epoch's committee on the destination chain before it can settle a message from that epoch.
Keeping that state current is a permissionless, on-chain-rewarded role in the bridge contracts rather
than something a Supra validator does.

### Native unbiasable randomness

`SUPRA_DKG` is what makes `supra_framework::randomness` usable. The block's randomness seed is derived
from the BLS threshold signature on the committed block's quorum certificate — which exists only once
validators hold threshold shares. That signature is deterministic and fixed only at the final round of
voting, which is what makes the seed unbiasable: no proposer can steer it by choosing what to propose,
and no subset of validators can steer it by choosing what to sign.

**No configuration step is needed to turn randomness on.** Seeding follows `SUPRA_DKG` directly.

#### Where the module comes from

`supra_framework::randomness` is Aptos's `randomness` module carried into the Supra framework. Apart
from the module and package names, the domain-separation tag and formatting, it is unchanged, so the
Move-level API is identical to Aptos's and **Aptos's own documentation for it applies directly**:
[Randomness on Aptos](https://aptos.dev/build/smart-contracts/randomness) treats how to design a
contract around these APIs at more length than this document does, and is worth reading alongside it.

What differs is underneath. The block seed the module hashes is produced by Supra's consensus and DKG,
as described above, rather than by the mechanism [AIP-41](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md)
specifies — and the module's own doc comments have not caught up. They still present AIP-41 as the
specification of the mechanism, still state unbiasability without noting the first-epoch case below,
and still describe the per-transaction counter as 32 bytes where the implementation uses 8. Those
comments are stale, not a description of different behaviour: the API and the guarantees set out in
this section are what the code does, and they will be revised in a future release. Until then, prefer
this section and the Aptos documentation over the comments in the module.

#### The API

Every function below is in `supra_framework::randomness`. Each draw is a domain-separated SHA3-256
hash over the block seed, the calling transaction's hash, and a counter that advances once per draw
within the transaction — so draws are independent of each other, and two transactions in the same
block get unrelated values from the same seed.

| Function | Returns |
|---|---|
| `u8_integer()`, `u16_integer()`, `u32_integer()`, `u64_integer()`, `u128_integer()`, `u256_integer()` | A uniform integer of that width |
| `u8_range(min_incl, max_excl)` and the `u16`/`u32`/`u64`/`u128`/`u256` equivalents | A uniform integer in `[min_incl, max_excl)` |
| `bytes(n)` | `n` uniform random bytes |
| `permutation(n)` | A uniformly random permutation of `[0, n)` |

Each call also emits a `RandomnessGeneratedEvent`, so randomness consumption is visible on chain.

Costs are worth knowing before you write a loop: one draw is one SHA3-256 hash, `permutation(n)`
costs `n − 1` draws, and `u256_range` costs two draws plus 256 modular additions. Draw once and reuse
rather than drawing per use.

#### The rules a developer must follow

**Randomness may only be consumed from a private or `public(friend)` entry function annotated
`#[randomness]`.** All three parts are required and each is enforced at a different point:

- The compiler rejects a non-public entry function that reaches randomness without the annotation, and
  rejects any *public* function that reaches randomness at all — unless it is annotated
  `#[lint::allow_unsafe_randomness]`, which exists to make the choice explicit and should not be used
  in code that handles value.
- Publishing rejects a `#[randomness]` annotation on a function that is not a non-public entry
  function.
- At runtime, a call from anywhere else aborts with `E_API_USE_IS_BIASIBLE` (code `1` in
  `0x1::randomness`).

The reason is that a caller who can wrap a randomness draw in their own code can inspect the result
and abort the transaction when they do not like it, retrying until they do. Restricting the entry
point to a function the module itself defines, and which no other module can call, removes the wrapper.

**Keep the gas cost of everything after the draw independent of the drawn value.** The test-and-abort
attack has a second form: if an unfavourable outcome costs more gas than a favourable one, a caller
can supply just enough gas that the unfavourable path runs out and the transaction reverts. Avoid
variable-cost data structures such as `SmartVector` and `SmartTable` on a post-draw path, avoid loops
whose length depends on the result, and avoid aborting on a particular outcome.

**On a network with no seed, a draw aborts rather than returning zeros.** If `SUPRA_DKG` is not active
— on a locally generated test network, for example — there is no block seed, and every randomness call
aborts with `0x1::option::EOPTION_NOT_SET` (`0x40001`). There is no view function a contract can use to
check first, so a module that must behave gracefully on such a network needs an off-chain check of
`0x1::randomness::PerBlockRandomness` before submitting, or its own on-chain switch.

**Two further notes.** The `*_range` functions reduce a 256-bit draw modulo the range, so their
uniformity is imperfect by a negligible margin; if you need exact uniformity, do rejection sampling on
`u256_integer()` yourself. And during the first epoch in which `SUPRA_DKG` is active the threshold keys
do not exist yet, so the seed for that epoch is weakly biasable; treat the epoch after activation as
the first one in which randomness carries its full guarantee.

---

## Confirming Activation

The `0x1::features::Features` resource is the authority on which flags are on. It holds a
`vector<u8>` bitset in which the flag with id `n` is bit `n % 8` of byte `n / 8`, least significant
bit first — so flags 97 to 100 are bits 1, 2, 3 and 4 of byte 12. Read it with
`GET /rpc/v3/accounts/0x1/resources/0x1::features::Features`.

A flag buffered by a governance proposal but not yet applied sits in `0x1::features::PendingFeatures`
instead, with the same encoding. Seeing a flag there and not in `Features` means the proposal has
executed and the epoch boundary that applies it has not yet been crossed.

Each flag also has an observable consequence, which is the better check that it is not merely enabled
but working:

| Flag | Observable signal |
|---|---|
| `SUPRA_BLS_KEYS` | `GET /rpc/v4/consensus/committees/{epoch}` returns a `V1` committee whose members carry the full key set |
| `SUPRA_BCFT_CERTIFICATES` | A certificate's `kind` reads `"BcftQuorum"` rather than `"Quorum"` |
| `SUPRA_DKG` | A `DKGStartEventV2` is emitted at the epoch boundary; `supra.dkg.phase.current` leaves `0`; commit QCs carry a `BlsThresholdSignature` |
| `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` | `GET /rpc/v4/transactions/certificates` returns certificates rather than `404`, and `include_proof=true` returns a non-null proof |
