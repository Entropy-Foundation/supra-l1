# REST API Change Log

Changes to the RPC node API since `supra_node_v11.3.6`.

## Changed Behavior

### View functions now execute under a bounded gas limit

`POST /rpc/v1/view`, `POST /rpc/v2/view`, `POST /rpc/v3/view`, `POST /rpc/v4/view`

**Breaking change.** View execution is now bounded by a maximum gas amount that defaults to
`2,000,000,000` gas units (the maximum gas allowed for a single transaction). Previously the
default was effectively unbounded (`u64::MAX`), and the v1 endpoint ignored the node's
configured limit entirely and always ran unbounded.

A view function whose execution exceeds the limit now fails with an out-of-gas error from the
Move VM instead of eventually returning a result. Clients that rely on expensive view
functions — large paginated table walks, aggregate computations over big collections — must
either split the call into smaller queries or ask their RPC provider to raise
`max_view_function_gas_amount` in `config.toml` (see the node configuration changelog).

View execution is also now dispatched to a bounded blocking thread pool shared by the API
(capacity `2 x CPU cores`). Under heavy concurrent view load, requests queue for a slot rather
than competing for the async worker threads. No new status code is returned; the observable
effect is latency, and unrelated endpoints are no longer starved by view traffic.

---

### Event page proofs are anchored to a single accumulator root

`GET /rpc/v4/events/{event_type}?include_proof=true`

All transaction-inclusion and event-emission proofs in one response page are now computed against
a single snapshot of the Move transaction accumulator. Previously the accumulator could advance
between events in the same page, so a page could contain proofs anchored to different roots and a
verifier checking the page against one root could see spurious failures. Response shape is
unchanged; verifiers may now resolve one root per page.

---

### Committee lookups resolve epochs that predate the node's upgrade

`GET /rpc/v2/consensus/committee_authorization/{epoch}`,
`GET /rpc/v4/consensus/committee_authorization/{epoch}`,
`GET /rpc/v4/consensus/committees/{epoch}`

These endpoints read the versioned authorized-committee table, which on nodes upgraded from
earlier releases contained only the epochs witnessed after that upgrade; earlier epochs were
reported as not found. On first start after upgrading, the node migrates the legacy committee
tables into the versioned one, so historical epochs still retained by the node now resolve
normally.

---

### Faucet funding transactions may be entry-function transfers

`GET /rpc/v1/wallet/faucet/{address}`

Faucet minter accounts may now be pre-funded accounts without the `SupraCoin` mint capability. A
minter of that kind funds requests with a `0x1::supra_account::transfer` entry-function
transaction rather than the mint-and-transfer Move script used by mint-capable minters. The
endpoint's request and response shapes are unchanged; clients that inspect the resulting funding
transaction must not assume a script payload.
