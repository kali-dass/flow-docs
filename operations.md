# Operations

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

Running Flow in production: process management, zero-downtime upgrades, and logging.

## Running

```bash
flow --config-kdl /etc/flow/config.kdl
```

Validate the config first — in CI, and again before any deploy:

```bash
flow --config-kdl /etc/flow/config.kdl --validate-configs
```

This parses the file, builds every route and connector, and exits without binding ports.

## Running in the background

```kdl
system {
    daemonize true
    pid-file "/var/run/flow.pid"
}
```

`pid-file` is **required** when `daemonize` is true, and must be an absolute path.

If you're running under systemd, Docker, or another supervisor, leave `daemonize false` and
let the supervisor manage the process — that's the usual choice.

---

## Zero-downtime upgrades

Flow can hand its listening sockets to a new process without dropping a connection. In-flight
requests on the old process drain to completion while the new one starts serving immediately.

> Graceful upgrade uses file-descriptor passing over a Unix socket and is **Linux only**.

### Setup

```kdl
system {
    upgrade-socket "/tmp/flow-upgrade.sock"
}
```

Note that `upgrade` itself is **not** a config file setting — it must be passed on the CLI.

### The handoff

The upgrade is a three-step handshake. Missing the third step is the most common mistake.

```bash
# 1. The old instance is already running. Find its PID.
pgrep -f 'flow.*config.kdl'

# 2. Start the new instance with --upgrade.
#    It connects to the upgrade socket and waits for the sockets to be handed over.
flow --config-kdl /etc/flow/config.kdl --upgrade &

# 3. Signal the old instance to hand over.
#    Do this promptly — the new process only waits about 6 seconds.
kill -SIGQUIT <old_pid>
```

On receiving `SIGQUIT`, the old process sends its listening file descriptors over the
upgrade socket, then gracefully drains and exits. The new process picks them up and takes
over.

### If it doesn't work

```
ERROR transfer_fd: No incoming socket transfer, sleep 1s and try again
ERROR transfer_fd: Giving up reading socket from: /tmp/flow-upgrade.sock, error: EAGAIN
```

This means the new process waited for the sockets and nobody sent them — **you didn't send
`SIGQUIT` to the old process**, or you sent it too late. Repeat the sequence and send the
signal within the ~6-second window.

Also note:

- **Signal the `flow` process itself.** If Flow is wrapped by a launcher script, `SIGQUIT`
  must reach the `flow` process, not the wrapper.
- **Linux only.** The file-descriptor transfer will not work on macOS.

### Docker

The graceful upgrade works by **replacing the running process** while keeping its listening
sockets open. That assumes a long-lived host process, which sits awkwardly inside a container
whose lifecycle is tied to PID 1 — when the old process exits, the container goes with it.

In containers, don't use `--upgrade`. **Roll the container instead**: start a new one, shift
traffic to it behind a load balancer, then retire the old one. That achieves the same
zero-downtime result using the orchestration you already have.

---

## Logging

Flow logs to stdout using `tracing`. Set the level with `RUST_LOG`:

```bash
RUST_LOG=info flow --config-kdl config.kdl
```

Startup logs report each service being configured, its listeners, and any warnings — for
example, if a connector sets a tuning option that its protocol doesn't use.

Warnings worth watching for at startup:

- `verify-cert=false` — certificate validation is disabled on a connector. Never in
  production.
- `allow-h2c=true` — the upstream hop is cleartext. Only acceptable on a trusted network.
- Notices that `max-h2-streams` or `h2-ping-interval-secs` is set on an HTTP/1.1-only
  connector, where it does nothing.

### The `timing-header` filter

`timing-header` adds latency headers to responses. It's cheap to run; gate it based on
whether you want to expose internal timing to clients, not for performance. See
[Policies](policies/timing-header.md).

### Planned: Comprehensive Logging & Observability

Flow's current logging is basic — startup messages and `RUST_LOG` level control only. Comprehensive observability features are **planned for future releases**:

- **Access logs** — structured JSON request/response logs (one per request), sampled or filtered to avoid overwhelming disk I/O
- **Metrics** — in-process histograms (latency, throughput, errors) exposed on a `/metrics` endpoint for Prometheus scraping
- **Tracing** — OpenTelemetry integration for distributed tracing across Flow and backend services
- **Log masking** — redaction of sensitive headers, tokens, and PII before logs reach disk

Until then, rely on:
- Response headers (via `timing-header` filter) for per-request latency
- `RUST_LOG=debug` for startup diagnostics
- External observability tools (load balancers, service mesh, APM agents) for fleet-wide visibility

---

## Sizing and capacity

Set `threads-per-service` to your **core count**. Hyperthreading does not help CPU-bound
proxy work.

```kdl
system {
    threads-per-service 2   // for a 2-core machine
}
```

### Always run with headroom

Flow's capacity limit is a **vertical cliff, not a gradual slowdown** — and crossing it costs
you three orders of magnitude in tail latency with no warning from the median. This is the
single most important thing to know when operating Flow.

Two operational rules follow from it:

- **Provision below the wall, not at it.** Leave real headroom — running at the edge means one
  traffic spike takes your p99 from milliseconds to seconds.
- **Alert on CPU headroom and high percentiles (p99/p99.9), never on median latency.** The
  median stays healthy right up to the moment the tail collapses, so a median-based alert will
  not fire until it is far too late.

**Measure your own workload** rather than relying on published figures — and run each test for
at least 10–15 minutes, because shorter runs overstate capacity.

See **[Performance](performance.md)** for the measured cliff, the capacity figures per
configuration, and guidance on benchmarking your own setup. Scaling guidance — adding cores,
and running multiple instances behind a load balancer — is there too.

---

## Health checking

Upstream health checking is **not yet implemented** (`health-check "None"`). Flow will
continue to send traffic to a failed backend. Until this lands, put a health-checking layer
in front of, or behind, Flow if you need automatic backend failover.

## Troubleshooting

### 502s on a connector with `proto="h2-only"`

`h2-only` requires the backend hop to be HTTP/2. If the backend does not negotiate HTTP/2
over TLS — it only speaks HTTP/1.1, TLS is misconfigured, or something between Flow and the
backend strips the `h2` ALPN — Flow returns **502** rather than silently proxying over
HTTP/1.1, and logs:

```
connector is h2-only but upstream negotiated HTTP/1.1 — refusing to proxy over H1
```

To resolve, pick one:

- **The backend should be HTTP/2** — fix it so it advertises `h2` in its TLS ALPN, then the
  connector works as configured.
- **HTTP/1.1 is acceptable for this backend** — change the connector to `proto="h2-or-h1"`,
  which uses HTTP/2 when the backend supports it and HTTP/1.1 otherwise.
- **The backend speaks cleartext HTTP/2** — use `proto="h2-only" allow-h2c=true` and no
  `tls-sni` (see [Configuration](configuration.md)).
