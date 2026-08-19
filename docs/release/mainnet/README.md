# Supra Mainnet

What mainnet is running today, and when each change took effect. Every value on this page is
recorded on chain — see [Verifying this yourself](#verifying-this-yourself) — so treat the chain,
not this page, as the authority if the two ever disagree.

Chain id `8`.

| | Version | Live since |
| --- | --- | --- |
| Move framework / VM | `aptosvm-v1.16_supra-v1.8.15` | epoch 7573, 2026-08-14 |
| Node binary | `supra_node_v11.5.1` | required on every node before the framework upgrade above |

Per-release changes are in the [release changelogs](../README.md).

## Feature flags

**All four of the v11 feature flags are active.**

| Flag | Id | Lifetime | Active since |
| --- | --- | --- | --- |
| `SUPRA_BLS_KEYS` | 97 | permanent | epoch 7575 |
| `SUPRA_BCFT_CERTIFICATES` | 98 | permanent | epoch 7575 |
| `SUPRA_DKG` | 99 | transient | epoch 7575 |
| `SUPRA_TRANSACTIONS_INCLUSION_PROOFS` | 100 | permanent | epoch 7575 |

Governance proposal `37` executed in epoch 7574 (2026-08-14 10:06 UTC) and buffered all four flags.
Buffered flags apply at the next epoch boundary, so all four became active together at **epoch
7575**. [What each flag changes](./v11.5.1/features.md) is described in the v11.5.1 release notes.

A prerequisite proposal (`36`, epoch 7573) published two resources mainnet genesis did not create
and installed the DKG configuration. Both were one-off and need nothing from operators.

### The flags flip together; their effects appear over three boundaries

The four bits are set in one step, but each feature engages at a different point, so the network's
observable behaviour changes over the following boundaries rather than all at once. Reading the
committee authorization for each epoch:

| Epoch | Committee | Certificate kind | Signature |
| --- | --- | --- | --- |
| up to 7574 | `V0` | `Validity` | `Ed25519Multisig` |
| 7575 | `V1` | `Validity` | `Ed25519Multisig` |
| 7576 | `V1` | `BcftQuorum` | `BlsMultisig` |
| 7577 onwards | `V1` | `BcftQuorum` | `BlsThresholdSignature` |

The v2 validator identity takes effect immediately at 7575. BCFT thresholds appear at 7576. The
first threshold signature produced by a DKG certifies epoch 7577 — the DKG that generated it ran at
the preceding boundary, which is why `SUPRA_DKG` takes one boundary longer to show up than the flag
that enabled it.

## What this means if you run a validator

**A validator whose consensus key is still a legacy bare ed25519 key is no longer eligible for the
active set.** Since `SUPRA_BLS_KEYS` became active, eligibility is re-tested at every epoch
boundary: a validator is retained only if it meets the minimum stake *and* carries a v2 consensus
key. One that does not is dropped from the active set — the same effect as falling below the
minimum stake — and becomes `INACTIVE` while keeping its stake.

The active set went from **83 validators in epoch 7574 to 66 in epoch 7575**, the boundary at which
the flag took effect. Any validator dropped then is still registered and still holds its stake, can
rejoin by the steps below:

To recover, rotate to a v2 key and re-join:

1. `rotate_consensus_key` with a v2 consensus key carrying its proof of possession (use `supra node identity rotate-keys`).
2. `join_validator_set` (use `supra move tool run` to call this function directly).

Eligibility is re-checked at the next epoch boundary, where the migrated key passes. Stake is
retained throughout; nothing is slashed or lost by having been ejected.

With `SUPRA_DKG` active, validators also take part in a distributed key generation round at each
epoch boundary. [DKG observability](../../operations/observability/dkg.md) covers what a healthy
round looks like and how to tell a slow one from a stuck one.

## Verifying this yourself

**Which flags are on.** `0x1::features::Features` is the authority:

```bash
curl -s 'https://rpc-mainnet.supra.com/rpc/v3/accounts/0x1/resources/0x1::features::Features'
```

It returns a `vector<u8>` bitset as hex, with no length prefix. Flag `n` is bit `n % 8` of byte
`n / 8`, least significant bit first — so flags 97 to 100 are bits 1 to 4 of byte 12. A flag that a
proposal has buffered but that has not yet been applied sits in `0x1::features::PendingFeatures`
instead, with the same encoding.

**Whether a feature is working, not merely enabled.** The committee authorization for an epoch shows
three of the four at once:

```bash
curl -s 'https://rpc-mainnet.supra.com/rpc/v4/consensus/committees/7577'
```

A `V1` committee means `SUPRA_BLS_KEYS`; a certificate `kind` of `BcftQuorum` means
`SUPRA_BCFT_CERTIFICATES`; a `BlsThresholdSignature` means `SUPRA_DKG` is producing output. The
per-flag signals, including the one for `SUPRA_TRANSACTIONS_INCLUSION_PROOFS`, are listed under
"Confirming Activation" in [features.md](./v11.5.1/features.md).
