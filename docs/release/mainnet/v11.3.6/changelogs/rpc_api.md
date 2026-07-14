# REST API — v11.3.4 Release Notes

Changes to node configuration files since `supra_node_v10.0.8`.

## New Endpoints

### API v4

A new API version has been introduced. The v4 endpoints are described in full below.

---

#### Block

**`GET /rpc/v4/block/height/{height}`**

Retrieves a block by height. Equivalent to v3, with one addition:

- New query parameter `include_proof` (boolean, optional). When `true`, each transaction in the block includes a `inclusion_proof` field — a Merkle proof authenticating the transaction's position in the executed transaction accumulator.

---

#### Consensus

The v2 consensus endpoints returned raw BCS binary bodies. The v4 equivalents accept content negotiation (`application/json`, `application/x-bcs`, `application/octet-stream`) and return a structured `AuthorizedCommittee` type when JSON is requested. Authentication is required for the `committee_authorization` and `block` endpoints; contact your RPC provider for access.

**`GET /rpc/v4/consensus/block`**
Get the latest consensus block.

**`GET /rpc/v4/consensus/block/height/{height}`**
Get the consensus block at the given height.

**`GET /rpc/v4/consensus/committee_authorization/{epoch}`**
Get the Committee Authorization for the given epoch as a structured `AuthorizedCommittee` object. Returns `404` if no authorization exists for the epoch.

**`GET /rpc/v4/consensus/committees/{epoch}`**
Get the Authorized Committee for the given epoch. Unlike the `committee_authorization` endpoint, this does not require authentication. Returns `400` if the epoch does not exist.

---

#### Events

**`GET /rpc/v4/events/{event_type}`**

Equivalent to v3, with one addition:

- New query parameter `include_proof` (boolean, optional). When `true`, each event in the response includes:
  - An event emission proof: a Merkle proof that the event was emitted by the stated transaction.
  - A transaction inclusion proof: a Merkle proof that the transaction is part of the executed transaction accumulator.

Both proofs use a Keccak256-based Merkle tree. A Solidity reference implementation for verifying these proofs is included in the endpoint description.

---

#### Proofs

A new endpoint category for cryptographic transaction and event proofs.

**`GET /rpc/v4/proofs/events/{event_hash}/transaction/{transaction_hash}`**

Returns an event emission proof for the given event within the specified transaction. This proves the event was emitted as part of that transaction.

- `400` — malformed hash.
- `404` — transaction hash not found.

**`POST /rpc/v4/proofs/events`**

Batch endpoint. Accepts a list of `(transaction_hash, event_types)` entries (up to 100 items, up to 500 resolved events total) and returns:

- A single **transaction inclusion certificate** at the node's latest certified block height, covering both Move and EVM accumulator roots. The certificate carries an aggregate signature from the consensus committee.
- One **transaction inclusion proof** per requested transaction, anchored to the appropriate accumulator root in the shared certificate.
- One **event emission proof** per matched event.

Because all proofs share a single certificate, a verifier can check the committee aggregate signature once and then verify each Merkle path against the shared root, rather than re-verifying the certificate once per event.

Both Move and EVM transactions may be included in the same request. For Move events, `event_types` entries are Move type tags (e.g. `0x1::coin::CoinDeposit`). For EVM events, they are canonical Solidity event signatures (e.g. `Transfer(address,address,uint256)`); the server keccak256-hashes the signature and matches it against each log topic.

- `400` — malformed request, batch too large, unrecognised transaction hash, unsupported transaction type, or invalid event type.
- `404` — no inclusion certificate is currently available on this node.

---

#### WebSocket

**`GET /rpc/v4/ws`** *(WebSocket upgrade)*

A Supra-native JSON-RPC 2.0 endpoint for subscribing to committed blocks. Implements `supra_subscribe` / `supra_unsubscribe`, mirroring the pattern used by the EVM endpoint at `/rpc/v1/eth`.

**Subscription topics**

- **`"newBlocks"`** — streams the full `BlockV4` for each committed block (same serialization as `GET /rpc/v4/block/height/{height}`). Accepts an optional params object:
  - `includeTransactions` (boolean, default `false`) — include the transaction list.
  - `includeProof` (boolean, default `false`) — include the transaction inclusion proof for each transaction.
- **`"newBlockHeaders"`** — streams only `BlockHeaderInfo` (height, hash, timestamp, etc.) without transactions.

**Protocol**

```json
// Subscribe to full blocks
{"jsonrpc":"2.0","id":1,"method":"supra_subscribe","params":["newBlocks",{"includeTransactions":true,"includeProof":false}]}

// Server acknowledges with a subscription ID (UUID)
{"jsonrpc":"2.0","result":"<subscription-id>","id":1}

// Server pushes a notification for every committed block
{"jsonrpc":"2.0","method":"supra_subscription","params":{"subscription":"<subscription-id>","result":{...BlockV4...}}}

// Unsubscribe
{"jsonrpc":"2.0","id":2,"method":"supra_unsubscribe","params":["<subscription-id>"]}
```

The endpoint respects the same `websocket_limits` authentication-token and connection-limit configuration as the EVM WebSocket endpoint. It can be disabled independently via the `websocket_limits.enable_supra_websocket` field in `config.toml` (see the node configuration changelog).

---

#### Transactions

**`GET /rpc/v4/transactions/{hash}`**

Equivalent to v3, with one addition:

- New query parameter `include_proof` (boolean, optional). When `true`, the response includes an `inclusion_proof` field — a Merkle proof for the transaction's position in the executed transaction accumulator.

**`GET /rpc/v4/transactions/certificates`**

Returns transaction inclusion certificates for a range of block heights. Both `start_height` and `end_height` are required query parameters.

These certificates record the consensus committee's aggregate signature over the Move and EVM transaction Merkle root accumulators at each block. Clients can use them to verify independently that a specific transaction was accepted by the Supra network.

- `404` — data not available for the requested range.
- `410` — data has been pruned. The `x-supra-oldest-block` response header indicates the oldest available height.

---

## Changed Behavior

### Simulate transaction — Move VM errors now return HTTP 200 (v3)

`POST /rpc/v3/transactions/simulate`

When a simulated transaction fails at the Move VM level, the response is now HTTP `200` with the failure details in the body, consistent with how finalized failed transactions are reported by the transaction lookup endpoints. Previously, some Move VM failures were returned as HTTP error responses.

### Move VM error messages are now human-readable

`POST /rpc/v3/view`, `POST /rpc/v3/transactions/simulate`

Error payloads for view and simulation calls previously rendered the `VMError` with `{:?}` Debug: the view `message` exposed Debug-formatted fields such as `sub_status: Some(393217)`, and the simulation `vm_status` wrapped its detail in a quoted, escaped `Some(\"…\")` string. Both now render the error through its `Display` form, so the `message` and `vm_status` fields read as plain text (for example `with sub status 393217`). The simulation `vm_status` keeps its single-line, comma-joined shape, but the detail is no longer Debug-wrapped — the status code and message are appended as plain text (`, Status Code: …, Message: …`) instead of the previous quoted, escaped `Some(\"…\")` blob. The Move/Rust backtrace is unaffected by this change: it is still appended when the node runs with `RUST_BACKTRACE` enabled. Clients that string-matched or parsed the previous format should update.

### Account resources for deleted objects (v3)

`GET /rpc/v3/accounts/{address}/resources`
Previously returned an error when the address was a deleted Move object. Now returns an empty array.

`GET /rpc/v3/accounts/{address}/resources/{resource_type}`
Previously returned an error when the address was a deleted Move object. Now returns `404`.

### Faucet balance cap

`GET /rpc/v1/wallet/faucet/{address}`

The faucet now enforces a maximum balance on the receiver. If the receiver already holds a balance at or above the configured cap, the request returns an error and no funding transaction is submitted.

### Corrected event object structure

Events returned by `/rpc/v3/events/{event_type}` now use an updated type for the `event` object within each list item:

- **`guid`** is now an object with two sub-fields — `creation_number` (string) and `account_address` (string) — rather than a plain string.
- **`sequence_number`** is now serialized as a string rather than an integer.

Events returned via v4 endpoints (e.g. from `/rpc/v4/events/{event_type}`) additionally include a **`hash`** field on each event. This is the keccak256 hash of the RLP-encoded event data and is the leaf value used in event emission proof verification.

### Genesis block handling

Block lookup endpoints at v3 (e.g. `GET /rpc/v3/block/height/{height}`) previously returned a `503 — archive still initializing` response when querying the genesis block (height 0) on a freshly started node. This has been corrected; the genesis block is now returned normally.

---

## Deprecations

The following endpoints were already deprecated before `supra_node_v10.0.8` and remain deprecated. Clients should migrate to their v4 equivalents.

| Deprecated endpoint | Replacement |
|---|---|
| `GET /rpc/v1/transactions/submit` | `POST /rpc/v4/transactions/submit` |
| `GET /rpc/v2/accounts/{address}` | `GET /rpc/v4/accounts/{address}` |
| `GET /rpc/v2/accounts/{address}/transactions` | `GET /rpc/v4/accounts/{address}/transactions` |
| `GET /rpc/v2/accounts/{address}/coin_transactions` | `GET /rpc/v4/accounts/{address}/coin_transactions` |
| `GET /rpc/v2/accounts/{address}/resources` | `GET /rpc/v4/accounts/{address}/resources` |
| `GET /rpc/v2/accounts/{address}/resources/{resource_type}` | `GET /rpc/v4/accounts/{address}/resources/{resource_type}` |
| `GET /rpc/v2/accounts/{address}/modules` | `GET /rpc/v4/accounts/{address}/modules` |
| `GET /rpc/v2/accounts/{address}/modules/{module_name}` | `GET /rpc/v4/accounts/{address}/modules/{module_name}` |
| `GET /rpc/v2/block` | `GET /rpc/v4/block` |
| `GET /rpc/v2/block/height/{height}` | `GET /rpc/v4/block/height/{height}` |
| `GET /rpc/v2/block/{block_hash}` | `GET /rpc/v4/block/{block_hash}` |
| `GET /rpc/v2/consensus/block` | `GET /rpc/v4/consensus/block` |
| `GET /rpc/v2/consensus/block/height/{height}` | `GET /rpc/v4/consensus/block/height/{height}` |
| `GET /rpc/v2/consensus/committee_authorization/{epoch}` | `GET /rpc/v4/consensus/committee_authorization/{epoch}` |
| `GET /rpc/v2/transactions/estimate_gas_price` | `GET /rpc/v4/transactions/estimate_gas_price` |
| `POST /rpc/v2/transactions/simulate` | `POST /rpc/v4/transactions/simulate` |
| `GET /rpc/v2/transactions/{hash}` | `GET /rpc/v4/transactions/{hash}` |
| `POST /rpc/v2/view` | `POST /rpc/v4/view` |

