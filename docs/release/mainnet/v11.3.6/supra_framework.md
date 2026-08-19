# Supra Framework Change Log

Changes to the Supra Move framework and gas schedule since `aptosvm-v1.16_supra-v1.7.28`.

This release ships framework tag `aptosvm-v1.16_supra-v1.8.9`. The thematic sections below
(feature flags, modules, gas schedule) describe the aggregate change through
`aptosvm-v1.16_supra-v1.8.6`; the tag-headed sections describe what each later tag added.

---

## `aptosvm-v1.16_supra-v1.8.6`

### `stake.move`

**New error code**

`EINVALID_PROOF_OF_POSSESSION` (24) — aborts when a submitted BLS multisig proof-of-possession does not verify against the submitted key.

**`initialize_validator` hardened; new `initialize_validator_genesis` entry point**

`initialize_validator` (operator/production path) now requires an appended BLS12-381 proof-of-possession in the consensus key blob. A new `initialize_validator_genesis` entry point (friend visibility) provides the same registration logic but is PoP-exempt — genesis output is trusted and carries no operator PoP.

Both functions delegate to a shared `initialize_validator_internal(…, genesis: bool)` helper.

**`validate_consensus_public_key` rewritten**

The function now returns the canonical key bytes (trailing PoP stripped) rather than `()`. Its logic is:

- *Legacy format (pre-v2 ed25519 key):* accepted without a PoP and stored unchanged.
- *New-format (`ValidatorPublicKeys`) — operator submission:* the trailing 96-byte BLS12-381 PoP is split off by `validator_public_keys::split_consensus_key_and_pop`, verified by `validator_public_keys::verify_bls_multisig_pop`, and the key bytes stored without it. `validate_static_keys` is called afterwards to reject blobs whose static keys fail on-curve / subgroup checks (deserialization alone does not run those checks).
- *New-format — genesis:* PoP is not required; key bytes stored verbatim.

PoP enforcement is tied to the key format, not the v2 feature flag, so a key registered during the pre-v2 migration window cannot skip it.

**`rotate_consensus_key` preserves DKG-managed threshold keys**

Under v2, an operator key rotation only replaces the four static keys (network, BLS multisig, class-group, ed25519) and leaves the DKG-managed BLS threshold keys (written by `set_dkg_output_keys`) intact. The merge is skipped (original behaviour preserved) before v2 is active, during genesis, or when the stored blob is empty (first rotation via `initialize_stake_owner`).

---

### `validator_public_keys.move`

**New error code**

`EINVALID_POP_LENGTH` (2) — aborts when the submitted blob is too short to contain a trailing BLS12-381 PoP.

**New constant**

`BLS12381_POP_NUM_BYTES = 96` — the byte length of a serialized BLS12-381 proof-of-possession (a G2 signature).

**New public functions**

| Function | Description |
|----------|-------------|
| `split_consensus_key_and_pop(blob)` | Splits a submitted key blob into its `ValidatorPublicKeys` bytes (prefix) and the trailing 96-byte BLS12-381 PoP. |
| `verify_bls_multisig_pop(pk, pop_bytes)` | Returns `true` iff `pop_bytes` is a valid BLS12-381 PoP for the BLS multisig key in `pk`. |
| `replace_static_keys(target, source)` | Overwrites the four static keys of `target` (network, BLS multisig, class-group, ed25519) with those from `source`, leaving all DKG-managed threshold key fields of `target` unchanged. |
| `validate_static_keys(pk)` | Re-validates the four static keys by running each key's validating constructor on the recovered bytes. Returns `false` if any key fails on-curve / subgroup / encoding checks. |

---

## New Feature Flags

Four new Supra-namespaced feature flags have been added to `features.move`. They must be enabled in the order shown — each flag is a prerequisite for the next.

| Flag | ID | Lifetime | Description |
|------|----|----------|-------------|
| `SUPRA_BLS_KEYS` | 97 | permanent | Enables the validator identity v2 format, which adds BLS consensus keys alongside the existing Ed25519 keys. Must be enabled before `SUPRA_DKG`. |
| `SUPRA_BCFT_CERTIFICATES` | 98 | permanent | Enables BCFT certificate threshold calculations. Must be enabled before `SUPRA_DKG`. |
| `SUPRA_DKG` | 99 | transient | Enables the DKG on-chain APIs. Requires both `SUPRA_BLS_KEYS` and `SUPRA_BCFT_CERTIFICATES` to be active first. |
| `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` | 100 | permanent | Enables generation of transaction inclusion proofs. |

---

## New Modules

### supra-framework

**`leader_ban_registry`**

Maintains the list of validators that have been banned for failing to propose a canonical block when elected. Interacts with `block`, `genesis`, and `reconfiguration`.

**`leader_ban_registry_config`**

On-chain configuration parameters for the leader ban registry (e.g. ban duration, probation thresholds).

**`supra_dkg`**

DKG on-chain state and helper functions. Manages DKG session lifecycle — start, key-share submission, and finalization — and is driven by `block` and `reconfiguration_with_dkg`.

**`dkg_committee`**

Represents the clan/family committee structure used by the DKG protocol, including `DkgCommittee` and `ReceiverCommittee` types.

**`validator_public_keys`**

Validator identity v2 representation. Stores a BLS12-381 consensus key and a class-groups public key alongside the existing Ed25519 network key. Also defines the BCFT certificate threshold types (`VALIDITY`, `QUORUM`, `UNANIMOUS`, `BCFT_VALIDITY`, `BCFT_QUORUM`).

**`dkg_config`**

On-chain configuration for DKG sessions.

### supra-stdlib

**`class_groups`**

Move-level interface to Supra class-groups cryptographic operations. Provides a `CGPublicKey` type and native functions for public-key deserialization and validation (`public_key_from_bytes`, `public_key_to_bytes`).

**`decode_bcs`**

Utilities for decoding BCS-encoded values from within Move.

---

## Modified Modules

### Validator identity and duplicate key prevention

**`stake.move`** — Major update. Adds support for the validator identity v2 format, integrating with `validator_public_keys` to store and validate BLS consensus keys. Validators are now prevented from registering duplicate network addresses or consensus keys; attempting to do so aborts the transaction.

### DKG integration

**`dkg.move` / `dkg.spec.move`** — Updated to use `DkgNodeConfig`; the DKG session lifecycle is now wired through `supra_dkg`.

**`reconfiguration_with_dkg.move`** — Reconfiguration flow updated to drive DKG sessions via `supra_dkg` rather than the base `dkg` module.

**`committee_map.move`** — Rebuilt to represent the clan/family committee structure required by the DKG protocol.

**`genesis.move`** — Now initialises `leader_ban_registry`, `supra_dkg`, `validator_public_keys`, and `dkg_config` at genesis.

**`block.move`** — Calls `leader_ban_registry` on every block to track whether the elected proposer successfully proposed a canonical block.

**`randomness_config.move`** — Minor updates to support DKG-gated randomness seeding.

### Governance

**`supra_governance.move`** — Governance transactions now gate randomness access on `features::supra_dkg_enabled()` (previously `consensus_config::validator_txn_enabled()`), allowing governance proposals to seed randomness once the DKG feature flag is active.

**`randomness.move`** — Exposed to governance transaction context.

### Automation

**`automation_registry.move`** — New public function `deconstruct_task_metadata_v2` that returns the task index as the first element of a 12-tuple. This extends the existing `deconstruct_task_metadata`; the original function is unchanged.

### Security fixes (pentesting audit)

**`fungible_asset.move`** — Access-control fixes identified during a security audit.

**`multisig_voting.move`** — Minor fixes from the security audit.

**`aggregator_v2.move`** — Minor fixes from the security audit.

**`committee_map.move`** — Additional boundary guards added as part of the audit remediation.

### Other

**`vesting_without_staking.move`**, **`pbo_delegation_pool.move`** — Updated to align with the validator identity v2 changes in `stake`.

**`config_buffer.move`**, **`reconfiguration.move`** — Minor additions to accommodate the new configuration modules.

---

## Gas Schedule Changes

### New gas feature version

`RELEASE_V1_16_SUPRA_V1_8_0` (version 25) is now `LATEST_GAS_FEATURE_VERSION`.

### New Supra stdlib gas parameters

New parameters for the `class_groups` native functions:

| Parameter | Value |
|-----------|-------|
| `class_groups.per_pubkey_deserialize.base` | 400,684 internal gas per arg |
| `class_groups.pop.base` | 206,000,000 internal gas |

---

## `aptosvm-v1.16_supra-v1.8.7`

### `stake.move`

**Behavioral change: validators still holding a legacy key are ejected from the active set.** Two
new helpers, `is_unrotated_legacy_key` and `is_eligible_active_validator`, are applied identically
when previewing the next validator set (`compute_next_validator_set_internal`) and when committing
it (`on_new_epoch`). A validator is retained only if it meets the minimum stake **and**, once the
validator-identity v2 format is enforced (feature `SUPRA_BLS_KEYS`, flag 97), still carries a v2
consensus key rather than a bare ed25519 key.

Operator impact: a validator that never rotated to a v2 consensus key before v2 was enforced is
dropped from the active set at the next epoch transition — the same effect as falling below the
minimum stake — and becomes `INACTIVE` while retaining its stake. Recovery is
`rotate_consensus_key` with a v2 key followed by `join_validator_set`; see
`aptosvm-v1.16_supra-v1.8.9` below, which is what makes that rotation succeed.

Applying the same test in both places keeps the DKG receiver committee and `set_dkg_output_keys`
(which read the preview) consistent with the committed consensus committee, and keeps
`validator_index` contiguous.

> **Confirm every validator has rotated to a v2 identity before the v2 format is enforced.** Those
> that have not are ejected at the following epoch transition.

---

## `aptosvm-v1.16_supra-v1.8.8`

### `supra_std::eth_trie`

**New constant.** `ETH_TRIE_ROOT_HASH_LENGTH = 32` — the required length of a trie root hash
(keccak256 / H256).

**Behavioral change.** `verify_eth_trie_inclusion_proof` and `verify_eth_trie_exclusion_proof` now
return "invalid proof" — `(false, vector[])` and `false` respectively — for a root that is not
exactly 32 bytes, before reaching the native. A wrong-length root cannot match any trie node, so it
is not a valid proof. These APIs remain gated on the existing `SUPRA_ETH_TRIE` feature flag.

### `native_verify_proof_eth_trie`

**Behavioral change.** The native returns `(false, vector[])` when the supplied root is not exactly
32 bytes, and when building the in-memory proof database fails. The `unwrap` on the database insert
was removed. A root of any other length is treated as an invalid proof rather than reaching code
that requires exactly 32 bytes.

---

## `aptosvm-v1.16_supra-v1.8.9`

### `stake.move`

**Behavioral change: an ejected validator can migrate its legacy key and rejoin.**
`rotate_consensus_key` now skips the DKG threshold-key merge when the *stored* key is an unrotated
legacy key, storing the incoming v2 blob verbatim instead.

This is what completes the recovery path opened by the ejection in `aptosvm-v1.16_supra-v1.8.7`:
the migration rotation previously aborted while trying to parse the stored legacy ed25519 key as a
`ValidatorPublicKeys` blob in order to merge threshold keys into it. An ejected validator can now
rotate to a v2 consensus key and call `join_validator_set`, rejoining the active set at the next
epoch transition with its stake intact.
