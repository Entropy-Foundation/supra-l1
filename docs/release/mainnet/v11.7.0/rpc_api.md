# REST API Change Log

Changes to the RPC node API since `supra_node_v11.5.1`.

## Changed Behavior

### A view function that fails now returns `400 Bad Request` instead of `500`

`POST /rpc/v1/view`, `POST /rpc/v2/view`, `POST /rpc/v3/view`

A view function can fail because the Move code it runs aborts, reads a missing resource, or
runs out of gas, or because the arguments supplied for it do not match its signature. All of
these describe the request, not the node, but each was previously answered with HTTP `500`.

They are now answered with HTTP `400`. Only a Move status that indicates a fault in the node
itself — a VM invariant violation — still returns `500`.

The response body reports the Move status the call failed with, and no longer carries the Rust
backtrace the VM attaches to it:

```json
{
  "message": "Failed to execute view function: ABORTED with sub status 4001 at location Module 0x5b8af9d0...::rfm_v2_randomness_router and message 0x5b8af9d0...::rfm_v2_randomness_router::borrow_dvrf_request at offset 43 at code offset 43 in function definition 2"
}
```

The `with sub status <n>` clause that carries the Move abort code is unchanged, so clients that
parse the abort code out of the message — `supra move tool view` among them — keep working.

**Client action.** Clients that treat `500` as "retry against another node" should treat `400`
from these endpoints as a permanent failure of that call, and not retry it.

### Move VM error bodies name the status instead of debug-printing an `Option`

`POST /rpc/v1/view`, `POST /rpc/v2/view`, `POST /rpc/v3/view`,
`GET /rpc/v1/accounts/{address}/modules/{module}` and the other module-reading endpoints

A failure the VM raised before reaching a location for it — an argument that does not match the
called function's signature, a module that fails to deserialize — was reported by Debug-printing
the optional message, so the body read `Some("...")`, or `None` when the failure carried no
message at all:

```json
{ "message": "Some(\"unexpected end of input\")" }
```

The body now names the Move status and carries the message unwrapped, matching the shape used
for the other Move failures:

```json
{ "message": "Move VM error: BAD_MAGIC and message unexpected end of input" }
```

These failures are still answered with HTTP `400`, with one exception: a Move status in the
invariant-violation range (`2000`-`2999`) describes a fault in the node rather than in the
request, so it is now answered with `500` and logged for the operator. `construct_args` reports
one as `INTERNAL_TYPE_ERROR`, and view-argument validation forwards it.

### Requesting event proofs for pruned data returns `410 Gone`

`GET /rpc/v4/events/{event_type}?include_proof=true`

The proofs of an event can only be built from transaction inclusion data that the node still
retains. Two cases that were previously served as an error are now reported as follows:

- If `start_height` is below the oldest block height that the node still holds inclusion data for,
  the request is answered with HTTP `410` and the `x-supra-oldest-block` response header carrying
  that height. This matches the behaviour of the other endpoints that serve pruned ranges.
- If the data behind an individual event becomes unavailable while the page is being built — the
  pruner can delete it at any moment — that event is returned with `"proofs": null` and the rest of
  the page is served as usual. `proofs` was already nullable, so no response shape changed.

Clients that require proofs should treat both cases as "this range is no longer provable" and query
a more recent range. Requests that do not set `include_proof` are unaffected: pruned inclusion data
does not prevent the events themselves from being served from the archive.
