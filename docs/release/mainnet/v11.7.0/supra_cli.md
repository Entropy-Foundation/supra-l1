# Supra CLI Change Log

Changes to the `supra` command-line tool since `supra_node_v11.5.1`.

---

## New Commands

### `supra config fetch-on-chain-config`

Reads the `SupraConfig` resource that a network is running and prints it as `OnChainParameters`. Previously nothing could read a config off a live network: the tooling could produce a blob (`supra config on-chain-config-serializer`) and decode one from a file (`supra config on-chain-config-deserializer`), but answering "what is Mainnet running?" meant knowing the resource address and type, knowing the REST route, `curl`-ing it, pulling the hex out of the JSON, writing that to a file, and only then running the deserializer over it.

The resource is requested as BCS and decoded through the same conversion a node applies to it, so the version reported is the version the network is actually running. The resource type is derived from the resource definition rather than spelled out in the command.

Network selection is the same as elsewhere in the CLI: `--network mainnet|testnet|localnet` fills in the endpoint, `--rpc-url` addresses a network that is not one of those three and conflicts with `--network`, and `--profile` takes the endpoint from a saved profile. With none of them given, the command prompts for an RPC URL.

`-o, --output-path` writes the raw blob to disk, so a running config can be diffed against a candidate blob or fed to other tooling. What it writes is the blob rather than the rendering, and it is written before the blob is parsed, so a config this version of the CLI cannot render is still on disk to inspect; the error says where it landed.

A network that does not carry the resource is reported as such, distinctly from an endpoint that is unreachable or serving something else.

| Flag | Description |
|---|---|
| `-o, --output-path <PATH>` | Path to write the raw `SupraConfig` blob to. Optional; the blob is printed as `OnChainParameters` either way. |
| `--network <NETWORK>` | Network to read from. Conflicts with `--rpc-url`, `--chain-id` and `--faucet-url`. |
| `--rpc-url <URL>` | RPC endpoint to read from. |
| `--profile <PROFILE>` | Take the endpoint from a saved CLI profile. Defaults to the active profile. |

---

## Fixes

### `supra profile migrate` — profiles with a rotated authentication key

Migrating a pre-v11 profile store now succeeds when it contains an account whose on-chain authentication key has been rotated. Previously the command failed with `Account private key is not valid for account <address>`, and because the whole store is converted before the database is created, one such profile blocked the migration of every profile in the file.

A pre-v11 profile records a single key alongside the account address. That key is the account's current signer, and it is also the key the address was derived from only until the account's authentication key is rotated; afterwards the original account key is not present in the profile at all. The migrated profile now records the account address and the authentication key, and leaves the account key unset, which is how the current CLI already represents a rotated account created with `supra profile new`.

No operator action is required, and profiles whose keys have not been rotated migrate exactly as before.

### One account may now have a profile on more than one network

A profile can now be created for an account address that another profile already uses, and for an authentication key another profile already uses, provided the two are on different chains. Previously the CLI database allowed each of those to appear on exactly one profile, so an account reachable on two networks — the core resources account `0xa550c18`, say, which exists on every one of them — could not have a profile for each. `supra profile new` rejected the second profile, and `supra profile migrate` failed the whole store with:

```
UNIQUE constraint failed: account_credentials.account_address
```

An account address identifies an account on one chain, so the same address regularly names an account on several networks, and one key may sign for it on all of them.

Two profiles for one account on the **same** chain remain rejected, now with a message naming the profile already using it:

```
Profile mainnet already uses account 0x…a550c18 on chain 8
```

That pair describes one account twice, and rotating either profile's authentication key would leave the other holding a key the chain no longer accepts. The chain is what distinguishes them, not the RPC endpoint: two profiles reaching one chain through different URLs are still two profiles for the same account.

This required a `cli_data.db` schema change, applied automatically by migration `0003` the first time this build opens an existing database: the `account_credentials` table is rebuilt without its uniqueness constraints on `account_address`, `authentication_private_key`, `account_private_key` and `staged_authentication_private_key`, and the one-account-per-chain rule is added as a trigger on `profile_records`. Profiles, their active selection and their network associations are preserved. No operator action is required.
