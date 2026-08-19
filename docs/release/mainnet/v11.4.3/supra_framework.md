# Supra Framework Change Log

Changes to the Supra Move framework and gas schedule since `aptosvm-v1.16_supra-v1.8.9`, the
framework tag released as v11.3.6.

This describes the net change across framework tags `aptosvm-v1.16_supra-v1.8.10` through
`aptosvm-v1.16_supra-v1.8.13`.

---

## New Feature Flags

None. The feature-flag set is unchanged from `aptosvm-v1.16_supra-v1.8.9`: no identifiers were
added to `std::features` (`features.move`) or to the Rust `FeatureFlag` enum. The only edits to
`features.move` are documentation comments on the existing `NEW_ACCOUNTS_DEFAULT_TO_FA_SUPRA_STORE`
(64) and `OPERATIONS_DEFAULT_TO_FA_SUPRA_STORE` (65) flags, recording that neither may be disabled
again once the coin-to-fungible-asset rollout is complete.

No new module is gated behind a flag, and none of the changes below requires a feature-flag
governance action. Every change lands with the framework upgrade itself.

---

## New Modules

None.

---

## Modified Modules

### `aptos_std::any`

**Breaking change.** `any::new(type_name, data)` is disabled. It now unconditionally aborts with
the new error code `EDISABLED` (2) via `error::invalid_state`, i.e. abort code `0x30002`.

Any client or third-party Move module that calls `any::new` will now abort. Values must be built
with `any::pack`, which records the caller's real type. `any::unpack` is unchanged.

### `aptos_std::from_bcs`

**New public function.** `to_vec_vec_u8(v: vector<u8>): vector<vector<u8>>` — BCS-decodes a byte
vector into a `vector<vector<u8>>`, using the type-checked `from_bytes` path.

### `supra_framework::config_buffer`

**Breaking change.** The public `config_buffer::extract<T>()` is deprecated and now aborts with the
new error code `EDEPRECATED` (2) via `error::unavailable`, i.e. abort code `0xD0002`. It is replaced
by `extract_v2<T>()`, which is `public(friend)`.

Every in-framework `on_new_epoch` was switched to `extract_v2`: `consensus_config`, `dkg_config`,
`evm_genesis_config`, `execution_config`, `gas_schedule`, `jwk_consensus_config`,
`leader_ban_registry_config`, `randomness_api_v0_config`, `randomness_config`,
`randomness_config_seqnum`, `supra_config`, `version`, `jwks`, `keyless_account`, and
`automation_registry`. Behavior of those handlers is otherwise unchanged.

### `supra_framework::code`

**New error codes**

| Code | Name | Meaning |
|------|------|---------|
| `0xB` | `EINVALID_METADATA_STRING` | Package metadata contains a `String` field whose bytes are not valid UTF-8. Raised via `error::invalid_argument`, so the abort code is `0x1000B`. |
| `0xC` | `EUNEXPECTED_EXTENSION` | Package metadata carries a populated `extension`, which is reserved for future use and must be empty. Raised via `error::invalid_argument`, so the abort code is `0x1000C`. |

**Breaking change.** `publish_package` now validates the supplied `PackageMetadata` before anything
is persisted, via a new internal `check_well_formed`. Package metadata arrives from
`publish_package_txn` as caller-supplied BCS and was previously stored unvalidated, so a publisher
could persist a `String` field holding non-UTF-8 bytes (which no Move code can construct) or an
`extension` field that is reserved and unused.

The invariants enforced are:

- `PackageMetadata.name` and `PackageMetadata.source_digest` hold valid UTF-8.
- Every `PackageDep.package_name` holds valid UTF-8.
- Every `ModuleMetadata.name` holds valid UTF-8.
- `PackageMetadata.extension` is `none`.
- Every `ModuleMetadata.extension` is `none`.

Toolchain-generated metadata satisfies all of these, so ordinary publishing is unaffected.
Publishers that hand-assemble metadata may now see `0x1000B` or `0x1000C`.

### `supra_framework::coin`

**Behavioral change: the `MigrationFlag` marker is no longer consulted.** The internal
`migrated_primary_fungible_store_exists` helper was removed. `is_account_registered`, `deposit` and
`force_deposit` now test for a paired primary fungible store with
`primary_fungible_store::primary_store_exists` alone.

Previously an account was only treated as having a usable fungible store if the store existed *and*
either the `NEW_ACCOUNTS_DEFAULT_TO_FA_SUPRA_STORE` feature was on or a `MigrationFlag` resource sat
alongside the store. An account whose primary fungible store was created by a direct FA deposit
(rather than by `register` / `migrate_to_fungible_store`) therefore carried no flag and was reported
as unregistered, so `register` would create a redundant `CoinStore` next to an existing FA balance
and `deposit` would route into the `CoinStore` rather than the store already holding funds.

Net effect for integrators:

- `is_account_registered<CoinType>` now returns `true` for an account that holds only a paired
  primary fungible store. `register` on such an account is now a no-op instead of creating a second,
  parallel balance.
- `deposit` / `force_deposit` route into the existing primary fungible store in that case.

The `MigrationFlag` struct is retained (unused) purely for upgrade compatibility.

**Behavioral change: `is_coin_store_frozen` no longer aborts on FA-only accounts.** When the
account is registered but has no `CoinStore<CoinType>` at all — i.e. it is registered only via a
paired primary fungible store — the function now returns `false` instead of aborting on the missing
resource. SUPRA/FA stores are not frozen through this legacy API.

### `supra_framework::multisig_voting`

**Breaking change.** Proposal success now requires that *approvals alone* reach the multisig
threshold. `get_proposal_state` for a closed proposal previously succeeded when
`yes_votes > no_votes && yes_votes + no_votes >= min_vote_threshold`, i.e. it counted NO votes
towards the threshold; a proposal with 2 approvals and 1 dissent passed a threshold of 3. The
condition is now `yes_votes >= min_vote_threshold && yes_votes > no_votes`.

### `supra_framework::stake`

**New error code**

`EUNEXPECTED_BLS_THRESHOLD_KEY` (25) — the submitted consensus public key carries a DKG-managed BLS
threshold key share. Raised via `error::invalid_argument`, so the abort code is `0x10019`.

**Breaking change: operator submissions may not carry DKG-managed threshold key shares.**
`validate_consensus_public_key` takes a new `stores_keys_verbatim` argument. On every path where the
submitted `ValidatorPublicKeys` blob becomes the stored consensus key unchanged, the blob must now
have all seven BLS threshold key-share fields empty.

The paths that now reject such a submission are:

- `initialize_validator` — registration stores the blob with no merge.
- The first `rotate_consensus_key` for a pool created via `initialize_stake_owner`, where the stored
  key is empty and there is nothing to merge into.
- The legacy-to-v2 migration rotation, where the stored key is a bare ed25519 key that cannot be
  parsed as a `ValidatorPublicKeys`.

A routine rotation of an already-migrated v2 key is exempt, because it takes the static-key merge
(`validator_public_keys::replace_static_keys`), which copies only the four static keys out of the
submission and discards its threshold fields. This matters in practice: an operator node
legitimately holds the current epoch's DKG shares at the moment it builds a rotation payload, so
that payload will contain them and is still accepted. Genesis output is trusted and is exempt, as it
already is from the proof-of-possession requirement.

**Documentation.** `set_dkg_output_keys` is now documented as the only writer permitted to populate
the stored BLS threshold key shares, and as rotating only the threshold types present in its
`committee_outputs` argument.

### `supra_framework::supra_governance`

**Behavioral change.** `reconfigure` now branches on `features::supra_dkg_enabled()` alone. It
previously also required `randomness_config::enabled()`. The `randomness_config` import was dropped.

Governance-initiated reconfigurations now start a DKG session whenever `SUPRA_DKG` is on.

### `supra_framework::validator_public_keys`

**New public function.** `has_no_bls_threshold_keys(pk: &ValidatorPublicKeys): bool` — returns
`true` only if all seven DKG-managed BLS threshold key-share fields are `option::none()`. This is
the check `stake` applies to submissions it stores verbatim.

**Behavioral change.** `validator_public_keys_from_bytes` now decodes via
`util::from_bytes<ValidatorPublicKeys>` instead of `any::new` + `any::unpack`. Same result, on a
type-checked path (`any::new` is now disabled).

### `supra_framework::supra_dkg`

**Behavioral change.** Deserialization of the aggregate commitment for all committees now uses
`util::from_bytes<OnChainAggregateCommitmentAllCommittees>` instead of `any::new` + `any::unpack`.
Same result, on a type-checked path.

### `supra_framework::util`

`validator_public_keys` and `supra_dkg` were added as friends, so they can use the type-checked
`util::from_bytes` in place of the now-disabled `any::new`.

### `supra_framework::automation_registry`

**New error code**

`EOWNER_NOT_ELIGIBLE_TO_RECEIVE_SUPRA` (50) — task owner is not eligible to receive SUPRA. (Code 49
is unused.)

**Breaking change: task owners must be able to receive SUPRA.** Task registration now asserts
`coin::is_account_registered<SupraCoin>(owner)` before doing any work. An owner account that cannot
receive SUPRA could not be paid its refunds, so registration is refused up front rather than
producing an unrefundable task. Note that, per the `coin` change above, an account holding only a
paired primary fungible store now satisfies this check.

**Behavioral change: refund accounting for short single-cycle tasks.** When unlocking fees for a
task being removed, the amount unlocked from the epoch-locked fees is now
`calculate_task_fee(..., cycle_info.start_time, ...)` — the fee for the task's actual active period
within the current cycle — instead of `calculate_automation_fee_for_interval` over the full cycle
duration. A single-cycle short-lived task is charged for the whole cycle but refunded on remaining
time to expiry, so unlocking the full-cycle fee did not correspond to what was actually charged.
The residual-time refund itself (`task_fee_for_residual_time / REFUND_FRACTION`) and the deposit
refund are unchanged. Epoch-locked fees exist only to stop the resource account being drained of
potential refunds within a cycle, and are reset and recalculated at the start of each cycle, so any
over-locked remainder only defers withdrawal for the current cycle.

### `supra_std::rlp`

**Behavioral change.** `decode_list_byte_array` now decodes the native's output with
`from_bcs::to_vec_vec_u8` instead of `any::new` + `any::unpack`. Same result, on a type-checked path
(`any::new` is now disabled). This API remains gated on the existing `SUPRA_RLP_ENCODE` feature
flag.

---

## Native Function Changes

These are changes to the Rust implementations behind existing Move natives. No native was added or
removed, and no Move signature changed.

### `algebra` hash-to-structure natives

**New abort code.** The `suite_from_ty_arg!` macro no longer unwraps `type_to_type_tag` on a
caller-supplied type argument. A type argument whose type tag cannot be constructed now produces an
ordinary Move abort with code `0x0B0063` (equivalent to `std::error::internal(99)`). The code matches
the one upstream Aptos uses for this fault.

### `vector_utils` sort natives

**Determinism fix.** `native_sort_vector_u64` and `native_sort_vector_u64_by_key` compute their gas
charge with integer arithmetic (bit length) rather than `f64::log2`. Gas charged during execution is
consensus-observable and `f64::log2` lowers to the platform's libm, which is not guaranteed to be
bit-identical across targets. The replacement is charge-preserving for every reachable vector
length, so no transaction's gas cost changes.

### `native_verify_proof_eth_trie`

The underlying `eth_trie` crate (0.4.0, unmaintained) is now vendored in-tree at
`third_party/eth_trie` so that two properties required for exposing it to caller-supplied input can
be fixed:

- Malformed node encodings are reported as ordinary errors in every case.
- The read and decode paths walk a caller-supplied structure of any depth iteratively instead of
  recursing once per level, as does the drop glue for a decoded tree.

---

## Gas Schedule Changes

None. `LATEST_GAS_FEATURE_VERSION` remains `RELEASE_V1_16_SUPRA_V1_8_0` (version 25). No gas
parameter was added, removed or re-priced — the `aptos-gas-schedule` crate is byte-identical between
the two tags. The `vector_utils` change above alters how a sort's gas charge is *computed*, not what
it comes to.
