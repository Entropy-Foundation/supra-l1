# Node Configuration Change Log

Changes to node configuration files since `supra_node_v11.3.6`.

---

## `smr_settings.toml` — Validator Node Settings

### New fields: `tokio_console_port` and `tcp_console_port`

Both fields are optional. When omitted (the default) the node binds an ephemeral port chosen at startup, preserving previous behaviour. Set them explicitly to bind the consoles on known ports — this also avoids a startup port-collision race on hosts running many node processes (e.g. CI), which could previously panic the node with `Address already in use`.

| Field | Type | Default | Description |
|---|---|---|---|
| `tokio_console_port` | integer (port) \| null | absent (random) | TCP port for the async-runtime `tokio-console` subscriber, bound on localhost. Connect with `tokio-console http://localhost:<port>`. |
| `tcp_console_port` | integer (port) \| null | absent (random) | TCP port for the node's `tcp_console` admin console (log-filter reload, network status, on-demand DB dump), bound on localhost. |

### Changed behaviour: missing `node.ws_server.certificates` files are now diagnosable from the log files

If a file named under `node.ws_server.certificates` (`cert_path`, `private_key_path`, or the optional `root_ca_cert_path`) is missing or unreadable, startup already failed and exited non-zero before this change — that part is unchanged. What was missing was any on-disk trace under `supra_node_logs/`: the failure never reached `supra.log` or `errors.log` (a logger-lifetime bug discarded the CLI's top-level error log line on every failed command, not just this one), and the message itself only reported the OS error (e.g. "No such file or directory (os error 2)"), not which of the (possibly three) configured paths was the problem. Operators saw a raw error on stderr but nothing in the log files they normally monitor, and it didn't say which file to check.

Now the same failure logs an `ERROR` to `supra_node_logs/{supra,errors}.log` naming the offending path, e.g. `failed to load TLS file "./ca_certificate.pem": No such file or directory (os error 2)`. No field changed shape; this only affects operators who deleted or misconfigured one of these paths.

---

## `config.toml` — RPC Node Settings

### New fields: `tokio_console_port` and `tcp_console_port`

Identical in structure and purpose to the same fields in `smr_settings.toml` (see above). Both are optional and default to a random port when omitted.

| Field | Type | Default | Description |
|---|---|---|---|
| `tokio_console_port` | integer (port) \| null | absent (random) | TCP port for the async-runtime `tokio-console` subscriber, bound on localhost. Connect with `tokio-console http://localhost:<port>`. |
| `tcp_console_port` | integer (port) \| null | absent (random) | TCP port for the node's `tcp_console` admin console (log-filter reload, on-demand DB dump), bound on localhost. |

### Changed default: `max_view_function_gas_amount`

The default gas budget for view-function execution (applied when the field is omitted from `config.toml`) changed from `u64::MAX` (effectively unlimited) to `2000000000`, matching the Aptos API default, which corresponds to the maximum gas allowed for a single transaction. A bounded default ensures every view-function call is metered, so no single call can consume execution capacity without limit. Nodes that set the field explicitly are unaffected.

View-function execution now also runs on the blocking thread pool under the API-wide concurrency limit rather than on the async worker threads, and the deprecated `/rpc/v1/view` endpoint respects this setting instead of a hard-coded unlimited budget.

Operators who rely on expensive view functions should set `max_view_function_gas_amount` explicitly rather than depending on the previous unlimited default.

| Field | Type | Old default | New default |
|---|---|---|---|
| `max_view_function_gas_amount` | integer | `u64::MAX` (unlimited) | `2000000000` |
