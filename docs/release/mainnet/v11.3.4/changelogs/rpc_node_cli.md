# `rpc_node` CLI — v11.3.4 Release Notes

Changes to the `rpc_node` command-line tool since `supra_node_v10.0.8`.

---

## New Commands

### `rpc_node replay`

Replays committed blocks from the node's reference chain store into a separate verifier database. Reads `config.toml` from `SUPRA_HOME` for path configuration.

| Flag | Description |
|---|---|
| `--start-height <HEIGHT>` | Inclusive start height. Defaults to the verifier's next executable height. |
| `--end-height <HEIGHT>` | Inclusive end height. Defaults to the reference MoveStore block height. |
| `--genesis-blob-path <PATH>` | Bootstrap the verifier database from genesis instead of a start height. Conflicts with `--start-height`. |
| `--verify-certificates <BOOL>` | Verify commit certificates during replay. Default: `true`. |
| `--compare-state-roots <BOOL>` | Compare state roots after replay. Default: `false`. |
| `--verify-reference-state <BOOL>` | Verify the full reference MoveStore state after replay. Conflicts with `--end-height`. Default: `false`. |
| `--verifier-db-suffix <SUFFIX>` | Suffix appended to chain-store and ledger DB paths for the verifier database. Default: `_replay`. |

---

## Changed Behavior

### Configuration file path

**Breaking change.** The node configuration file is now read from `$SUPRA_HOME/config.toml`. Previously it was read from a hard-coded path that could not be overridden at runtime.

A new global flag, `--supra-home <DIR>`, sets the home directory for any subcommand, taking precedence over the `SUPRA_HOME` environment variable.

---

### Logger initialisation

The logger is now initialised in `main` for every subcommand — immediately after argument parsing, before the subcommand runs — so panics and errors during startup of any subcommand are captured to disk. Previously, the logger was started only inside the `start` subcommand.

---

### `rpc_node start`

**Breaking change.** The short flags `-i` (for `--info-to-stdout`) and `-e` (for `--warn-err-to-stderr`) have been removed. Use the long forms only.
