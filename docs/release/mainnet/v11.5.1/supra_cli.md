# Supra CLI Change Log

Changes to the `supra` command-line tool since `supra_node_v11.4.3`.

---

## New Commands

### `supra config on-chain-config-deserializer`

Deserializes a `SupraConfig` blob and prints it as `OnChainParameters` — the inverse of `supra config on-chain-config-serializer`. Previously there was no way to check what a blob actually carried: reviewing a governance proposal, or confirming which version a network is running, meant decoding the BCS bytes by hand.

The blob is parsed through the same conversion a node applies to the on-chain `SupraConfig` resource, trying V2, then V1, then V0, so the version reported here is the version the network would adopt, and a blob this command rejects is one that governance cannot apply.

The input file may hold the blob in any of the forms the tooling and the chain produce, and the format is detected automatically:

- raw bytes;
- a `0x`-prefixed hex string, as returned when reading the resource from the chain;
- a list of decimal byte values — both the JSON array that `on-chain-config-serializer` writes and the whole Move script that `generate-on-chain-config-update-script` emits, so a generated proposal can be fed straight back in to see what it would apply. The `vector[..]` literal is located within the script rather than the file being scanned for digits, so the parameter block the script records above it is not mistaken for part of the blob.

| Flag | Description |
|---|---|
| `-p, --path-to-serialized-config <PATH>` | Path to the file holding the serialized `SupraConfig` blob. Required. |

---

## Changed Behavior

### `supra config on-chain-config-serializer` and `supra config generate-on-chain-config-update-script` — new `--config-version` flag

Both commands previously always emitted a `SupraConfig` **V1** blob, carrying only the `[mempool]` and `[moonshot]` sections of `genesis_parameters.toml`. The `[commitments]` and `[dkg]` sections could not be delivered to a running network at all, so a network's DKG timing parameters were fixed at whatever its binary compiled in, and producing a V2 blob meant patching the CLI or hand-assembling the BCS bytes.

`--config-version <v1|v2>` selects the version. It **defaults to `v2`**, which is what genesis has emitted since v11.3.1, so an invocation that does not pass the flag now produces a different blob than it did before this release. Pass `--config-version v1` to reproduce the previous output.

**A network still running V1 adopts all four sections at once when a V2 blob is applied.** Both `TransactionsInclusionCertifier` and `CertificatesSynchronization` fall back to the same value on a V1 network, and both now read the same on-chain field once a V2 blob is applied:

| Consumer | V1 fallback | Value under V2 |
|---|---|---|
| Proposal / certificate-sync retry delay (`TransactionsInclusionCertifier`) | 500 ms | `commitments.proposal_retry_delay_ms` (default `500`) |
| Missing-certificates request interval (`CertificatesSynchronization`) | 500 ms | `commitments.proposal_retry_delay_ms` (default `500`) |

So applying a V2 blob with default `[commitments]` settings to a V1 network is a no-op for both consumers. Set `[commitments]` and `[dkg]` in the config directory deliberately before generating a V2 blob if you do want either to change, rather than letting them fall back to the defaults.

The version is echoed in the command's stdout, and `generate-on-chain-config-update-script` records the full parameter set in a comment at the top of the generated Move script, so a reviewer can see which variant a proposal carries.

| Flag | Default | Description |
|---|---|---|
| `--config-version <VERSION>` | `v2` | `SupraConfig` version to emit. `v1` carries `[mempool]` and `[moonshot]`; `v2` additionally carries `[commitments]` and `[dkg]`. |

### `supra node identity rotate-keys` and `supra node identity update-network-address` — pre-v11 identity files are migrated automatically

Both commands now migrate a `node_identity.pem` that is still in the pre-v11 format before reading it. Previously only `supra node smr run` did, so an operator who upgraded the binary and ran either rotation command against an unmigrated identity got a serde error with no indication of the cause or the remedy:

```
Error: Validator identity error: Failed to read or write the identity store: Failed to deserialize identity: missing field `id` at line 1 column 4230
```

Migration is idempotent and a no-op for an identity that is already current, so it costs nothing on an up-to-date profile. Both commands accept an already-migrated identity exactly as before.

`--simulate` does **not** migrate, so that a dry run still leaves the identity on disk untouched. Simulating against a pre-v11 identity reports the file and stops; migrate with a real run first.

Migration now takes a backup first. The existing file is copied to `node_identity.<RFC3339 timestamp>` alongside it and the path is printed to stderr before anything is rewritten, so the original keys survive an interrupted migration. This applies to `supra node smr run` as well, which has always performed the migration.

Migration preserves the network address, network key, and consensus key, but **generates fresh BLS and class group keys** and rewrites `validator_public_identity.json`. The command now says so explicitly on completion. If you have already distributed your public identity file, redistribute it.

`supra genesis sign-supra-committee` deliberately does **not** migrate. It only reads the identity, and the fresh keys would leave the signature inconsistent with the `supra_committees.json` that the other operators have already built from the old public identity. It reports the pre-v11 file and stops, leaving the decision to migrate and restart the ceremony with the operator.

### Validator identity errors name the file, and `SUPRA_HOME` is logged

Errors from the encrypted identity store now include the path they were reading, and a pre-v11 identity file is reported as such rather than as a raw deserialization failure. A file that is neither format still surfaces the underlying serde error, so genuine corruption is not misreported as merely old.

Every `supra` invocation now logs the resolved `SUPRA_HOME` and which input it came from — the `--supra-home` flag, the `SUPRA_HOME` environment variable, the working directory when it is itself named `.supra`, or the default `.supra` relative to the working directory. The last two depend on where the process was started, so a CLI invocation and the validator it is meant to act on can read different files without either one saying so; this makes that visible. It is emitted at `INFO`, so it lands in `supra_node_logs/supra.log` and appears on stdout only under `--info-to-stdout`.
