# `supra` CLI — v11.3.4 Release Notes

Changes to the `supra` command-line tool since `supra_node_v10.0.8`.

This is a large, security-focused release of the `supra` CLI. The headline theme is **how the CLI handles credentials**: profiles now live in an encrypted database instead of a key file, passwords are resolved and verified up front (with optional OS-keyring storage), and the ability to pass raw private keys on the command line has been removed. Alongside that, the command surface has been tidied — read-only queries no longer accept signing flags, several commands were renamed for clarity, and network selection is now a single `--network` preset.

Because of these changes, **most scripts and automation that drive the v10.0.8 CLI will need updates**. Start with the [Migration checklist](#migration-checklist--operators-ci--qa); the precise, per-command details follow.

> Compatibility: this release targets the same on-chain protocol as the v11.3.1 node. Commands that build transactions are network-compatible with current Supra mainnet, testnet, and local networks.

---

## Highlights

- **Encrypted profile database.** CLI profiles move from `smr_secret_key.pem` to an encrypted SQLCipher database, `cli_data.db`. Use `supra profile migrate` to carry existing profiles over.
- **Up-front password handling.** Passwords are resolved and validated during argument parsing — before any work begins — from a cache, environment variable, OS keyring, or an interactive prompt, in that order. Failures are reported immediately instead of mid-command.
- **OS keyring integration.** A new `supra keyring` command stores passwords in the platform keyring (Secret Service, Keychain, or Credential Manager) so nodes and scripts can run without prompts.
- **No more raw keys on the command line.** Signing is done through the active profile (`--profile`); the `--private-key`, `--private-key-file`, and related flags have been removed from every transaction command.
- **Safer read-only commands.** Query and faucet commands (`balance`, `view`, `download`, `verify-proposal`, …) no longer accept transaction-signing flags and can target any address with `--account`.
- **New diagnostics & operations.** `supra node smr replay` replays committed blocks into a verifier database; `supra move account rotate-key` rotates an account's on-chain authentication key.
- **Cleaner output and home layout.** A global `--stdout text|json|none` controls command output, and all CLI artifacts now live under a single `SUPRA_HOME` directory (default `./.supra`).

---

## Migration checklist — operators, CI & QA

This release changes credential handling, the command surface, and the on-disk layout. Work through this checklist before upgrading node-operator scripts, CI pipelines, and test harnesses:

1. **Pin your home directory.** Files are no longer written to the current working directory. When `SUPRA_HOME` is unset and `--supra-home` is not given, the CLI now uses a `./.supra` subdirectory. Set `SUPRA_HOME` (or pass `--supra-home <DIR>`) so commands read and write the location you expect, and drop any reliance on `SUPRA_LOG_DIR` (logs now live under `SUPRA_HOME`).
2. **Migrate your profiles.** Run `supra profile migrate` once per home directory to move profiles from `smr_secret_key.pem` into `cli_data.db`. It is a no-op if there is nothing to migrate, or if `cli_data.db` already exists.
3. **Stop passing raw keys.** Remove `--private-key`, `--private-key-file`, `--account-address`, and `--profile-path` from transaction commands. Select the signer with `--profile`, use `--simulate` for dry runs, and set endpoints with `--network` / `--rpc-url` / `--chain-id`.
4. **Make runs non-interactive.** Replace `--assume-yes` / `--assume-no` with `--accept-defaults`. For unattended jobs, populate the OS keyring (`supra keyring store cli-profile` / `node-identity`) or set the password / `*_RESET` / `*_KEY` environment variables — passwords are now resolved and validated at parse time, so a missing or wrong one fails fast instead of mid-command.
5. **Swap environment variables.** `NODE_OPERATOR_KEY_PASSWORD` is no longer read — use `CLI_PROFILE_PASSWORD` and `NODE_IDENTITY_PASSWORD`. See the [environment variable reference](#environment-variables).
6. **Rename commands in automation.** `supra node identity rotate-consensus-key` → `rotate-keys`; `rotate-network-address` → `update-network-address` (and drop `-n` / `--new-key`). `supra nidkg` has been removed. `--path-to-smr-settings` → `--config-dir`. Never pass `--rpc-url` to `fund-with-faucet` — use `--faucet-url` or a `--network` preset.
7. **Re-key weak passwords.** Strength (zxcvbn score ≥ 3) is now enforced when *unlocking* an artifact on the normal path, not just when setting one. If an existing password is weaker than that, ordinary commands reject it at parse time — set a stronger one with `supra profile reset-password` or `supra node identity reset-password`, which intentionally skip the strength gate so a weak legacy password can authorize its own replacement.
8. **Check your RPC endpoints.** Chain-ID autofill now calls `GET /rpc/v3/transactions/chain_id`; point automation at nodes that serve `/rpc/v3`.
9. **Update shell completion.** Use `eval "$(supra --completion bash)"` — completion is a global flag, not a subcommand.

---

## Breaking changes

### Home directory and file locations (`SUPRA_HOME`)

All CLI artifacts are resolved relative to a single `SUPRA_HOME` directory.

- The default home changed. Previously, when `SUPRA_HOME` was unset, files were written directly into the current working directory. The default is now a `.supra` subdirectory of the current working directory, created automatically if it does not exist.
- The global `--supra-home <DIR>` flag sets the home directory for any subcommand and takes precedence over the `SUPRA_HOME` environment variable.
- `SUPRA_LOG_DIR` is no longer read; logs always live under `SUPRA_HOME`.
- A `SUPRA_HOME` that resolves to a symbolic link is rejected.

### Profile storage moved to an encrypted database

CLI profiles were stored in the encrypted file `smr_secret_key.pem`. They are now stored in an encrypted SQLCipher database, `cli_data.db`, inside `SUPRA_HOME`.

`supra profile migrate` reads the old `smr_secret_key.pem` and writes the profiles into `cli_data.db`. It is a no-op when no `smr_secret_key.pem` is present, and it will not overwrite an existing `cli_data.db`.

### Raw signing flags removed from transaction commands

The following flags have been removed from every command that sends or simulates a transaction — including `move tool run`, `move tool publish`, `move tool run-script`, `move automation register/cancel/stop-tasks`, `move multisig approve/create/create-transaction/execute/execute-reject/execute-with-payload/reject`, `move account transfer/create/create-resource-account/rotate-key`, `governance propose/vote/approve-execution-hash/execute-proposal`, `node identity rotate-keys/update-network-address`, and the `node delegation-pool` subcommands:

- `--private-key`
- `--private-key-file`
- `--account-address` (on signing commands)
- `--profile-path`
- `--assume-yes` and `--assume-no`

Signing is now done exclusively through the active CLI profile, selected with `--profile`. `--sender-account` remains available to specify a different sender address, and `--rpc-url` and `--chain-id` remain available as per-command network overrides.

Three flags were added to these commands in their place:

- `--network <mainnet|testnet|localnet>` — selects a network preset, setting both the RPC URL and chain ID at once. (It conflicts with explicit `--rpc-url`/`--chain-id`/`--faucet-url`.)
- `--accept-defaults` — accepts the default value for every interactive prompt without confirmation (replacing `--assume-yes`/`--assume-no`).
- `--simulate` — runs the transaction against the RPC node without committing it.

### Read-only queries no longer accept signing flags

Query, view, and faucet commands no longer accept transaction-build or signing flags, and most can now target any address.

- **`supra move tool show`, `list`** — gained `--account <ADDRESS>` to query any account without making it the active profile. `show` additionally accepts `--resource-type` as a visible alias for `--name`.
- **`supra move tool view`, `download`** — are now read-only. They dropped `--private-key`, `--private-key-file`, `--account-address`, `--profile-path`, `--sender-account`, `--gas-unit-price`, `--max-gas`, `--expiration-secs`, `--assume-yes`, `--assume-no`, and `--status`, and gained `--network` and `--accept-defaults`. They have no `--simulate` flag, as they send no transaction. On `download`, `--account` is now optional (it was required in v10.0.8) and defaults to the active profile's address.
- **`supra move account balance`, `fund-with-faucet`** — gained `--account <ADDRESS>` and became read/faucet-only, dropping the same signing flags listed above and gaining `--network` and `--accept-defaults`. `fund-with-faucet` now **rejects an explicit `--rpc-url`** with an actionable error; select the faucet endpoint with `--faucet-url` or a `--network` preset. (Previously `--rpc-url` was accepted but ignored, so a request could fund the profile's configured network instead of the intended one.)
- **`supra move multisig verify-proposal`** — is now read-only, dropping the same signing flags and gaining `--network` and `--accept-defaults`. It has no `--simulate` flag. (`--chain-id` was already present in v10.0.8.)

### `supra profile` changes

- **`new`** — the positional `[PRIVATE_KEY]` argument has been removed. To import an existing account, supply the key via `CLI_PROFILE_AUTHENTICATION_KEY` or `CLI_PROFILE_ACCOUNT_KEY`. `--account-address` remains (its help was clarified — it denotes the address of an imported account that has rotated its on-chain authentication key), and `--accept-defaults` was added for non-interactive use. A profile now records an *account key* (from which the address is derived) separately from an *authentication key* (used to sign); the two differ once an account has rotated its on-chain authentication key.
- **`sign`** — dropped `--private-key`, `--private-key-file`, `--account-address`, `--rpc-url`, `--faucet-url`, `--chain-id`, `--sender-account`, and `--profile-path`. The remaining flags are `--profile`, `--file`, `--stdin`, and `--expected-signature`. When `--expected-signature` is supplied and does not match, the command now exits non-zero with an error; previously it printed `Signature is incorrect` and still exited `0`. The comparison ignores case and an optional `0x` prefix; a match still prints `Signature is correct`.
- **`list`** — the `-l` short alias was removed. An optional `[NAME]` argument was added to inspect a single profile, and `--reveal-secrets` now displays secrets in the terminal's alternate screen rather than on standard output.
- **`modify`** — `--account-address` was removed; a profile's account address is immutable after creation. `modify` accepts `--network`, `--chain-id`, `--rpc-url`, `--faucet-url`, and `--accept-defaults`.
- **`--profile-path` removed everywhere** — the flag is gone from all profile subcommands (`new`, `activate`, `remove`, `rename`, `reset-password`, `sign`, `list`, `modify`, `migrate`). The corresponding `--identity-path` flag was removed from `supra node identity reset-password`. All artifact paths now resolve relative to `SUPRA_HOME`.

### `supra node identity` subcommands renamed

| v10.0.8 | v11 |
|---|---|
| `supra node identity rotate-consensus-key` | `supra node identity rotate-keys` |
| `supra node identity rotate-network-address` | `supra node identity update-network-address` |

`rotate-keys` rotates all static keys atomically. The key format changed in v11 with the introduction of the DKG protocol and the move to BLS multisignatures and threshold signatures; `--delegation-pool-address` remains a required argument.

`--private-key`, `--private-key-file`, and `--profile-path` were removed from both commands; `--simulate`, `--network`, and `--accept-defaults` were added. On `update-network-address`, the v10.0.8 `-n, --new-key` flag was also removed — the network key is now managed with the other static keys via `rotate-keys`.

After a successful on-chain rotation, the CLI signals the running validator to apply the new network key immediately; all other key changes take effect at the start of the next epoch. Both commands require two passwords: the CLI profile password is resolved first (to load the signing keys from `cli_data.db`), followed by the node identity password (to unlock the validator identity file). Each is resolved in the standard order.

### `supra node smr run`

The `--disable-dynamic-identity-loading` flag has been removed; dynamic identity loading on epoch changes is now always active. This follows from the DKG protocol, which generates fresh private key shares for each validator every epoch.

The validator password is resolved at parse time (from the environment variable, OS keyring, or an interactive prompt) and cached for the lifetime of the process — there is no interactive prompt at runtime. After updating the password with `supra node identity reset-password`, restart the node to pick it up, unless the keyring is in use, in which case no restart is needed.

### `supra config` flag rename

For `supra config on-chain-config-serializer` and `supra config generate-on-chain-config-update-script`, the required flag `--path-to-smr-settings` was renamed to `--config-dir` (the short forms `-p` and `-s` are unchanged). Both commands now read `genesis_parameters.toml` from the supplied config directory instead of reading an `SmrSettings` file directly.

### Global flags

| v10.0.8 | v11 |
|---|---|
| `-o <LOG>` (log level) | Removed. |
| `-i` (short for `--info-to-stdout`) | Removed; use `--info-to-stdout`. |
| `-e` (short for `--warn-err-to-stderr`) | Removed; use `--warn-err-to-stderr`. |

`--info-to-stdout` and `--warn-err-to-stderr` remain as long flags and, like the other global flags, are now accepted on every subcommand. See [Output format and verbosity](#output-format-and-verbosity) for `--stdout` and the new `--verbose` flag.

### Removed

- **`supra nidkg`** — the command group has been removed.
- **`NODE_OPERATOR_KEY_PASSWORD`** — the environment variable, deprecated in v10.0.8 in favor of `CLI_PROFILE_PASSWORD` and `NODE_IDENTITY_PASSWORD`, is no longer read.
- **`SUPRA_LOG_DIR`** — no longer read; logs live under `SUPRA_HOME`.
- The raw-key and `--assume-yes`/`--assume-no` flags listed above, and the per-command `--profile-path` / `--identity-path` flags.

---

## New commands and features

### `supra keyring`

Manages passwords stored in the operating-system keyring (Secret Service on Linux, Keychain on macOS, Credential Manager on Windows) for non-interactive operation. Entries are scoped to the active `SUPRA_HOME`.

| Subcommand | Description |
|---|---|
| `supra keyring store <cli-profile\|node-identity>` | Store a password in the OS keyring. |
| `supra keyring remove <cli-profile\|node-identity>` | Remove a stored password from the OS keyring. |
| `supra keyring list` | List passwords currently stored in the OS keyring. |

`supra profile new` stores the new profile password in the OS keyring automatically when a keyring is available, so subsequent commands run without prompting.

### Shell completion (`--completion`)

`supra --completion <shell>` prints a shell completion script for `bash`, `zsh`, `fish`, `powershell`, or `elvish`, then exits. It is a global flag on the root command (not a subcommand):

```sh
eval "$(supra --completion bash)"
```

### `supra node smr replay`

Replays committed blocks from the node's reference chain store into a separate verifier database, reading `smr_settings.toml` from `SUPRA_HOME` for path configuration.

| Flag | Description |
|---|---|
| `--start-height <HEIGHT>` | Inclusive start height. Defaults to the verifier's next executable height. |
| `--end-height <HEIGHT>` | Inclusive end height. Defaults to the reference MoveStore block height. |
| `--genesis-blob-path <PATH>` | Bootstrap the verifier database from genesis instead of a start height. Conflicts with `--start-height`. |
| `--verify-certificates <BOOL>` | Verify commit certificates during replay. Default: `true`. |
| `--compare-state-roots <BOOL>` | Compare state roots after replay. Default: `false`. |
| `--verify-reference-state <BOOL>` | Verify the full reference MoveStore state after replay. Conflicts with `--end-height`. Default: `false`. |
| `--verifier-db-suffix <SUFFIX>` | Suffix appended to chain-store and ledger DB paths for the verifier database. Default: `_replay`. |

### `supra move account rotate-key`

Rotates the on-chain authentication key for the Move account of the active CLI profile. The account address is preserved, and on success the profile's authentication key is updated in `cli_data.db`.

The new private key is read from `CLI_PROFILE_AUTHENTICATION_KEY` when set; otherwise it is entered interactively. This command resolves the CLI profile password **even under `--simulate`**, because it must read the account's current signing key from `cli_data.db` to build and sign the rotation-proof challenge.

### Output format and verbosity

- `--stdout <text|json|none>` — a global flag controlling command output: `text` (default) is human-readable, `json` emits one JSON document per command for scripting and SDK use, and `none` suppresses command output. The human-readable value `display-trait` was renamed to `text`, and the flag is now accepted on every subcommand (it was root-only before). The `json` and `none` modes carry over unchanged from v10.0.8. Completion scripts are always emitted verbatim regardless of this setting.
- `--verbose` — a new global flag that prints extra diagnostic output.

### Read-only `--account` targeting

`supra move tool show` / `list` and `supra move account balance` / `fund-with-faucet` accept `--account <ADDRESS>` to operate against any address without changing the active profile. On `supra move tool download`, `--account` is now optional and defaults to the active profile's address.

---

## Password and credential handling

### Resolution order

Passwords that unlock an existing artifact are resolved and validated during argument parsing, before any command executes, in this order:

1. A value already cached during the same invocation.
2. The relevant environment variable, validated against the encrypted artifact.
3. The OS keyring.
4. An interactive prompt — up to five attempts, and only when a TTY is available.

If a password is present but incorrect, the CLI falls through to the next source. If no valid source is found and no TTY is available, the command exits at parse time with a clear error.

Commands run with `--simulate` do not require a password — the one exception is `supra move account rotate-key` (see above).

### Strength and length

Passwords are capped at 1,024 characters. Strength (zxcvbn score ≥ 3) is enforced both when setting a new password and when unlocking an existing artifact on the standard path (cache, environment variable, keyring, or prompt). The strength check is skipped only on the reset and migration flows — `supra profile reset-password`, `supra node identity reset-password`, and `supra profile migrate` — so a weak legacy password can still authorize its own replacement.

### Environment variables

The variables marked **New** did not exist in v10.0.8. `CLI_PROFILE_PASSWORD` and `NODE_IDENTITY_PASSWORD` carry over from v10.0.8 with the same names, but are now resolved and validated at parse time.

| Variable | Purpose | New in v11 |
|---|---|---|
| `CLI_PROFILE_PASSWORD` | Unlocks the CLI profile database. | No |
| `NODE_IDENTITY_PASSWORD` | Unlocks the validator identity file. | No |
| `CLI_PROFILE_PASSWORD_RESET` | New password for `supra profile reset-password`. | **New** |
| `NODE_IDENTITY_PASSWORD_RESET` | New password for `supra node identity reset-password`. | **New** |
| `CLI_PROFILE_AUTHENTICATION_KEY` | Authentication private key for `supra profile new`; the new key for `supra move account rotate-key`. | **New** |
| `CLI_PROFILE_ACCOUNT_KEY` | Account private key for `supra profile new`. | **New** |

`NODE_OPERATOR_KEY_PASSWORD` is no longer read (see [Removed](#removed)).

---

## Other behavioral changes

### Chain ID resolution uses `/v3`

The CLI now resolves a network's chain ID via `GET /rpc/v3/transactions/chain_id` instead of the older `/v1` endpoint. All current Supra networks expose `/v3`. An RPC node that serves only `/v1` or `/v2` will fail chain-ID autofill — for example during `supra profile new` / `supra profile modify`, or whenever a custom `--rpc-url` is supplied without `--chain-id`.

### `supra genesis sign-supra-committee`

The validator identity password is now resolved and validated during argument parsing rather than mid-execution, and the command requires an existing validator identity. If none is present it fails fast at parse time with `Validator identity not found … Generate one with 'supra node identity new'`, and an incorrect password is reported before any signing work begins. The password is resolved in the standard order (`NODE_IDENTITY_PASSWORD`, OS keyring, interactive prompt).

---
