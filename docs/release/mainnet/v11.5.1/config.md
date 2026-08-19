# Node Configuration Change Log

Changes to node configuration files since `supra_node_v11.4.3`.

---

## `smr_settings.toml` — Validator Node Settings

### Changed validation: `dkg_thread_pool_size` rejects `0` and is clamped to the logical core count

`dkg_thread_pool_size` sizes the thread pool the DKG uses for compute-heavy work, chiefly verifying the dealings that every other validator broadcasts at an epoch boundary. Two values previously behaved in ways an operator would not expect:

- **`0` was accepted and meant "one thread per logical core"** — the widest possible pool, the opposite of the throttle implied by writing `0`. This is the underlying thread-pool library's convention for "choose automatically". Startup now fails with `dkg_thread_pool_size must be at least 1. Omit the field to use the number of physical CPU cores.`
- **A value above the host's logical core count was used as written.** Since the work is CPU-bound, the surplus workers only added contention — most visibly when a settings file written for a wide machine was redeployed onto a narrower one. The value is now clamped to the logical core count and a `WARN` naming both numbers is logged. The node still starts.

Neither change affects a node that omits the field (still the physical core count) or sets it to any value between `1` and the logical core count.

The guide now also documents the field's interaction with the node's two other CPU consumers — the async runtime and the Move execution pool, both sized to the logical core count and neither configurable — since the three are sized independently and their sum can exceed the host's cores. See "Sizing `dkg_thread_pool_size`" in the node configuration guide.

