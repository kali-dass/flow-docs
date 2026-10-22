# Configuration Reference

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Routing](routing.md) • [Policies](policies.md) • [Performance](performance.md)

Flow is configured with a single [KDL](https://kdl.dev) file, passed with `--config-kdl`.
The file has two top-level blocks: `system` and `services`.

```kdl
system {
    // process-wide settings
}

services {
    MyService {
        // listeners, routes, connectors
    }
}
```

`config/example-config.kdl` in the repository is a working, heavily-commented reference.

## Validating

Always check a config before deploying it:

```bash
flow --config-kdl myconfig.kdl --validate-configs
```

Flow parses the file, builds every route and connector, and exits — without binding ports.
Errors point at the exact line.

---

## How many times can each item appear?

Most settings and blocks may appear **once** in their scope; a few are **lists** that
repeat. Setting a "once" item twice is a config error — Flow rejects it at load and points at
the second occurrence.

| Scope | Appears **once** | Can **repeat** |
|---|---|---|
| Top level | `system`, `services` | — |
| `system` | `threads-per-service`, `daemonize`, `pid-file`, `upgrade-socket`, `ca-file`, `real-ip-source` | — |
| `system.trusted-proxies` | — | `cidr` |
| `system.logging` | `app-log` | — |
| `system.logging.app-log` | `level`, `output`, `log-dir` | — |
| `system.logging.access-log` | *(reserved — content ignored)* | — |
| `system.dns-resolver` | `ndots`, `timeout-secs`, `attempts`, `ip-version`, `empty-answer-threshold` | `nameserver`, `search-domain` |
| `services` | — | named service blocks |
| A service | `listeners`, `routes`, `path-control`, `route-cache-dynamic-capacity`, `keepalive-timeout-secs` | — |
| `listeners` | — | listener address lines |
| `routes` | — | `route` blocks |
| A route | `match`, `connectors`, `allowed-methods`, `allowed-protocols`, `path-control`, `cache-response` | — |
| `match` | the matcher | sub-matchers inside `all` / `any` |
| `connectors` | `load-balance` | backend address lines |
| A connector | each attribute (`proto`, `tls-sni`, `max-h2-streams`, timeouts…) — each once | — |
| `load-balance` | `selection` (with required `key` attribute for `"FNV"`/`"Ketama"`), `discovery`, `health-check`, `health-check-path`, `health-check-interval-secs`, `health-check-timeout-secs`, `unhealthy-threshold`, `healthy-threshold`, `circuit-breaker { … }` | — |
| `circuit-breaker` | `error-threshold-pct`, `window-secs`, `min-requests`, `cooldown-secs`, `probe-requests`, `window-strategy`, `bucket-count` (only with `window-strategy "TimeBuckets"`) | — |
| `path-control` | `request-filters`, `upstream-request`, `upstream-response` | — |
| A filter phase | — | `filter` nodes |

Two things that look like exceptions but aren't:

- **`allowed-methods` / `allowed-protocols` appear once but take many values** —
  `allowed-methods "GET" "POST"`, one line with several arguments. Two separate
  `allowed-methods` lines is an error.
- **To require several conditions, use `all` / `any`, not repeated `match`** — a `match` block
  holds one matcher; nest conditions under `all`/`any` inside it. See [Routing](routing.md).

---

## `system` block

Process-wide settings.

```kdl
system {
    threads-per-service 2
    daemonize false
    pid-file "/tmp/flow.pidfile"
    upgrade-socket "/tmp/flow-upgrade.sock"
    // ca-file "./config/ca/upstream-ca.crt"

    logging {
        app-log {
            level "info"
            output "stdout"
            // output "file"
            // log-dir "/var/log/flow"
        }
    }

    // Real client IP — required when Flow sits behind a proxy or CDN.
    // trusted-proxies {
    //     cidr "10.0.0.0/8"
    // }
    // real-ip-source "xff"   // or "header:CF-Connecting-IP" etc.

    // dns-resolver { ... }   // see DNS discovery below
}
```

| Setting | Meaning |
|---|---|
| `threads-per-service` | Worker threads per service. Set this to your **core count**. Hyperthreading does not help a CPU-bound proxy. |
| `daemonize` | Run in the background. If `true`, `pid-file` **must** be set. |
| `pid-file` | Absolute path to the PID file used when daemonizing. |
| `upgrade-socket` | Absolute path to the Unix socket used for zero-downtime upgrades. See [Operations](operations.md). |
| `ca-file` | A CA bundle trusted for **all** outgoing upstream TLS connections. Use when every upstream shares one private CA. For per-upstream CAs, use `ca-cert-path` on the connector instead. |
| `real-ip-source` | How to resolve the effective client IP. `"peer"` (default), `"xff"` (X-Forwarded-For), or `"header:<name>"` (single named header). Requires `trusted-proxies` when not `"peer"`. See [Real client IP](#real-client-ip) below. |

### `logging.app-log`

Flow's own application log — startup, config validation, warnings, errors, upgrade events.
Not per-request access logging (Flow doesn't have that yet).

| Setting | Meaning |
|---|---|
| `level` | One of `error`, `warn`, `info`, `debug`, `trace`. Defaults to `info`. Overridable per-invocation with `--log-level`. |
| `output` | `"stdout"` (default) or `"file"`. |
| `log-dir` | A **directory** (not a file path) Flow writes `flow-application.log` into. Required when `output` is `"file"`; rejected if set while `output` is `"stdout"`. |

Use `output "file"` when you background Flow with `--daemonize` — otherwise logs keep going to
stdout, which has nowhere useful to go once the process is detached. See
[Operations](operations.md#running-in-the-background).

> `upgrade` is deliberately **not** settable in the file — it must be passed on the CLI.

> The **license file** is also not part of this block — like `upgrade`, it's CLI/environment
> only (`--license-file` / `FLOW_LICENSE_FILE`). See [Licensing](license.md).

### `logging.access-log` (reserved)

An `access-log` block may appear inside `logging` — Flow accepts it without error, but does
not yet read or act on anything inside it. Per-request access logging is not implemented.
The name is reserved so configs that include it now will not need to be changed when the
feature ships.

### Real client IP

By default Flow uses the raw TCP peer address as the client IP for all IP-based policies
([ip-restriction](policies/ip-restriction.md), [rate limiting](policies/rate-limiting.md)
`key-from="source-ip"`, and load-balancing hashing). When Flow sits behind a proxy, CDN, or
cloud load balancer, that peer address is the proxy's address — not the real client's.

Configure `trusted-proxies` and `real-ip-source` to tell Flow where to find the real IP:

```kdl
system {
    trusted-proxies {
        cidr "10.0.0.0/8"              // your LB / proxy CIDR range
    }
    real-ip-source "xff"               // read X-Forwarded-For
}
```

For CDN deployments, use the CDN's proprietary header:

```kdl
system {
    trusted-proxies {
        cidr "103.21.244.0/22"         // Cloudflare IP range (example)
    }
    real-ip-source "header:CF-Connecting-IP"
}
```

| CDN / proxy | `real-ip-source` value |
|---|---|
| Generic (X-Forwarded-For) | `"xff"` |
| Cloudflare | `"header:CF-Connecting-IP"` |
| AWS CloudFront | `"header:CloudFront-Viewer-Address"` |
| Fastly | `"header:Fastly-Client-IP"` |
| Akamai | `"header:True-Client-IP"` |
| Generic Nginx proxy | `"header:X-Real-IP"` |

**`real-ip-source` requires `trusted-proxies`** — Flow refuses to start without it when source
is anything other than `"peer"`. This is a security requirement: a header-based source without
a trusted-proxy list lets any client inject an arbitrary IP and bypass IP-based policies. Only
connections arriving from a declared trusted CIDR will have their forwarded header trusted;
all other connections use the raw TCP peer regardless.

`trusted-proxies` without `real-ip-source` is accepted (the CIDRs are stored but have no effect
until `real-ip-source` is set).

### Forwarded headers written to upstream

Flow automatically writes `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Host` to
every upstream request. This is always-on and requires no configuration.

| Header | Value |
|---|---|
| `X-Forwarded-For` | Resolved client IP, appended to any existing trusted chain |
| `X-Forwarded-Proto` | `https` or `http` — the protocol of the downstream connection |
| `X-Forwarded-Host` | The original `Host` header from the client request |

**Strip-before-append**: if the TCP peer is not in `trusted-proxies`, any incoming
`X-Forwarded-*` headers are stripped before Flow appends its own values. This prevents a
client from pre-populating the chain to spoof its source IP.

---

## `services` block

Each named child of `services` is an independent proxy service with its own listeners and
routes.

```kdl
services {
    Example1 {
        route-cache-dynamic-capacity 10000
        keepalive-timeout-secs 60

        listeners { ... }
        routes { ... }
        path-control { ... }   // optional, applies to every route
    }
}
```

| Setting | Default | Meaning |
|---|---|---|
| `route-cache-dynamic-capacity` | 10000 | Max entries in this service's dynamic route cache. Only routes with regex or complex matchers use this cache — prefix-only routes use a separate static index. Increase if you have a large number of distinct request paths and observe cache evictions in the metrics. For most deployments the default is sufficient. |
| `keepalive-timeout-secs` | 60 | How long an idle **downstream** (client) H1 connection is held open waiting for the next request. `0` means no cap — defer entirely to the client's `Keep-Alive` header. No effect on H2 connections. |

---

## Listeners

A listener is a socket Flow accepts client connections on. The node name is the bind
address; TLS is enabled by supplying a certificate and key.

```kdl
listeners {
    "0.0.0.0:8000"
    "0.0.0.0:8443" cert-path="./certs/test.crt" key-path="./certs/test.key" offer-h2=true
}
```

| Attribute | Meaning |
|---|---|
| `cert-path` | PEM certificate. Enables TLS on this listener. |
| `key-path` | PEM private key. Required with `cert-path`. |
| `offer-h2` | Offer HTTP/2 to clients on this listener. Works with or without TLS. |

Unix domain sockets are also supported.

### Offering HTTP/2

`offer-h2=true` means **offer HTTP/2 to clients on this listener** — Flow works out how:

- **With TLS**, HTTP/2 is negotiated via ALPN. HTTP/1.1 clients still connect normally.
- **Without TLS**, clients may use **cleartext HTTP/2 (h2c)**. Flow detects it per connection,
  so the *same* listener still serves HTTP/1.1 clients too.

Test a plain h2c listener with:

```bash
curl --http2-prior-knowledge -v http://localhost:8000/   # HTTP/2
curl -v http://localhost:8000/                            # HTTP/1.1
```

#### Two rules to know

> **1. All plain HTTP listeners in one service must agree on `offer-h2`** — set it on all of
> them or none. A mixed set is rejected when the config loads.

> **2. Cleartext HTTP/2 and TLS cannot share a service.** If any plain listener sets
> `offer-h2`, that service must not also have a TLS listener. This combination is rejected
> when the config loads.
>
> The reason: cleartext HTTP/2 is a **service-wide** setting, not a per-listener one. Turning
> it on for a plain listener would also force HTTP/2 onto that service's TLS connections,
> which breaks HTTPS clients still using HTTP/1.1. Flow rejects the config rather than serve
> it incorrectly.

#### When one service has both plain and TLS listeners

This is the common shape, and rule 2 constrains it. Given a service with a plain listener and
a TLS listener, these are your options:

| Plain listener | TLS listener | Valid | What you get |
|---|---|---|---|
| *(omit `offer-h2`)* | *(omit `offer-h2`)* | ✅ | Plain: HTTP/1.1 · TLS: HTTPS/1.1 |
| *(omit `offer-h2`)* | `offer-h2=true` | ✅ | Plain: HTTP/1.1 · TLS: HTTP/2, falling back to HTTPS/1.1 |
| `offer-h2=true` | *(either)* | ❌ | Rejected at load — see rule 2 |

**In short: the TLS listener may set `offer-h2` freely; the plain listener may not, so long as
they share a service.** The second row is usually what you want:

```kdl
listeners {
    "0.0.0.0:8000"                                            // HTTP/1.1
    "0.0.0.0:8443" cert-path="…" key-path="…" offer-h2=true   // HTTP/2 via TLS + HTTPS/1.1
}
```

**If you need cleartext HTTP/2 *and* TLS**, use two services — one for each:

```kdl
services {
    Internal {
        listeners {
            "0.0.0.0:8000" offer-h2=true                      // h2c + HTTP/1.1
        }
        routes { /* … */ }
    }
    Public {
        listeners {
            "0.0.0.0:8443" cert-path="…" key-path="…" offer-h2=true
        }
        routes { /* … */ }
    }
}
```

Each service needs its own `routes` block — configuration is not shared between services. Note
that separate services also maintain separate connection pools to your backends, which is worth
knowing if both point at the same upstream.

> ⚠️ Cleartext HTTP/2 is **unencrypted**, exactly like plain HTTP/1.1. Only offer it where plain
> HTTP is already acceptable.

### HTTP/2 connection concurrency and memory

Each downstream HTTP/2 connection is capped at **100 concurrent streams** (requests in flight
on that connection). This is a fixed, sane default — most clients never approach it.

If a client needs more concurrency than one connection's cap allows, it opens **additional**
connections rather than being throttled — this is standard HTTP/2 client behavior, not an
error. Each open connection has a small, fixed memory cost on the Flow side. Under normal
traffic (a handful of connections per client) this is negligible. A client that opens **very
many** simultaneous connections — for example an aggressive load-testing tool pushed well past
its needs — will proportionally increase Flow's memory footprint. This is a property of how
many connections are open, not how many requests per second are flowing through them.

If you are capacity-planning for a workload with unusually high connection counts, budget
roughly **1 MB per open HTTP/2 connection** in addition to Flow's baseline footprint (typically
tens of MB).

---

## Routes

Each `route` has a `match` block, the methods and protocols it accepts, and the connectors
it forwards to.

```kdl
routes {
    route {
        match {
            path-prefix "/echo"
        }

        allowed-methods "GET" "PUT" "PATCH" "DELETE" "HEAD" "OPTIONS"
        allowed-protocols "http" "https" "h1" "h2"

        connectors { ... }
        path-control { ... }            // optional, per-route
        cache-response { ... }          // optional, per-route — see Cache response policy
        downstream-read-timeout "None"  // optional — "None" or positive integer (seconds)
        downstream-write-timeout 30     // optional — positive integer (seconds)
    }
}
```

**All routes must live in a single `routes { }` block.** A service has one `routes` block
containing many `route` entries — not several `routes` blocks. Writing two is rejected at
load (see [How many times can each item appear?](#how-many-times-can-each-item-appear)).

### `allowed-methods` (required)

Every route must list the HTTP methods it accepts. A request with a method that isn't
listed does not match the route (and returns **502** if nothing else matches). Method names
are case-insensitive; non-standard methods like `PURGE` work.

### `allowed-protocols` (required)

Every route must list at least one of:

| Value | Meaning |
|---|---|
| `http` | Plain HTTP |
| `https` | HTTP over TLS |
| `h1` | HTTP/1.1 |
| `h2` | HTTP/2 |

A request arriving on a protocol that isn't listed is rejected with **426 Upgrade Required**
(plain HTTP where TLS is required) or **400** (wrong HTTP version).

To accept TLS + HTTP/2 only, and reject TLS + HTTP/1.1:

```kdl
allowed-protocols "h2"
```

See [Routing](routing.md) for the `match` block and how competing routes are scored.

### Downstream timeout reference

Both downstream timeout fields are **H1-only** — they have no effect on HTTP/2 downstream connections.

| Field | Default | H1 | H2 | Values |
|---|---|---|---|---|
| `downstream-read-timeout` | 60 s | ✅ | no-op | `"None"` (disable), positive integer (seconds) |
| `downstream-write-timeout` | none | ✅ | no-op | positive integer (seconds) |

### `downstream-read-timeout` (optional)

Controls the downstream H1 read timeout for this route. Pingora's default is **60 seconds** — correct for normal REST routes but too short for long-lived streaming connections (SSE, WebSocket, MCP).

| Value | Meaning |
|---|---|
| absent | Pingora default (60 s) |
| `"None"` | Disabled — no read timeout; required for SSE, WebSocket, MCP |
| positive integer | Explicit cap in seconds (e.g. `3600`) |

```kdl
downstream-read-timeout "None"   // SSE / WebSocket / MCP — unbounded
downstream-read-timeout 3600     // generous-but-bounded (1 hour)
```

`0` is rejected. No-op on HTTP/2 downstream connections. See [Non-REST Protocols](non-rest-protocols.md)
for full SSE and WebSocket configuration examples.

### `downstream-write-timeout` (optional)

Controls the downstream H1 write timeout for this route. Pingora sets **no write timeout by default** — a stalled client can hold the connection open indefinitely. Set a positive integer to cap how long Flow waits for the client to accept a write.

| Value | Meaning |
|---|---|
| absent | No write timeout (Pingora default) |
| positive integer | Explicit cap in seconds (e.g. `30`) |

```kdl
downstream-write-timeout 30    // drop clients that stall for > 30 s on a write
```

`0` is rejected. No-op on HTTP/2 downstream connections. See [Non-REST Protocols](non-rest-protocols.md)
for streaming protocol configuration examples.

---

## Connectors (upstreams)

Connectors are the backends a route forwards to. The node name is the upstream socket
address; everything else is an attribute.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        discovery "Static"
    }

    "10.0.0.1:3000" tls-sni="backend.internal" proto="h2-or-h1"
    "10.0.0.2:3000" tls-sni="backend.internal" proto="h2-or-h1"
}
```

### Multiple backends

List as many backend addresses as needed — Flow distributes traffic across all of them according to the `selection` strategy:

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        health-check "Active"
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
    "10.0.0.3:3000"
}
```

With DNS discovery, a single hostname that resolves to multiple IPs works the same way — Flow maintains the pool automatically as the address changes (see [DNS as a load balancer](dns.md#multiple-addresses-dns-as-a-load-balancer)):

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        discovery "Dns"
        health-check "Active"
    }
    "orders-svc.default.svc.cluster.local:8080"
}
```

### Load balancing

| `selection` | Order / behavior |
|---|---|
| `"RoundRobin"` | Cycles through backends in the order they are listed in the config — request 1 → first backend, request 2 → second, and so on, wrapping around. Default. |
| `"Random"` | Picks a backend at random for each request. No guaranteed order. |
| `"FNV"` | FNV hash of the selector key — the same key always maps to the same backend. Sticky, but remaps all keys when the pool size changes. |
| `"Ketama"` | Consistent hash of the selector key — sticky and stable under membership change. When a backend is added or removed, only a fraction of keys remap. Preferred over `"FNV"` when backends join or leave frequently. |

**When to use which:**
- Homogeneous backends with no session state → `RoundRobin`
- Simple even spread without ordering guarantees → `Random`
- Session affinity (e.g. cache locality, MCP session stickiness) → `Ketama`
- Session affinity on a stable, rarely-changing pool → `FNV`

`"FNV"` and `"Ketama"` require a `key` attribute — Flow rejects the config without one. `key` controls what is hashed:

```kdl
selection "Ketama" key="UriPath"
```

| `key` | Hashed on |
|---|---|
| `UriPath` | Request URI path |
| `SourceAddrAndUriPath` | Client IP + URI path. Uses the resolved client IP — set `real-ip-source` for consistent hashing behind a proxy. |

### Discovery

`discovery` controls how Flow builds and maintains the upstream backend pool.

| `discovery` | Behavior |
|---|---|
| `"Static"` | Fixed set of backends for the life of the process. IP addresses are used as-is. Hostname connectors are resolved **once at startup** via the OS resolver and only the **first A record** is kept — subsequent records and any future DNS changes are ignored until restart. No background work. |
| `"Dns"` | Periodic DNS re-resolution. Flow resolves the connector hostname on startup and re-resolves it at the interval set by `dns-interval-secs` (default `30`). The backend pool is updated automatically when the address changes — no restart needed. |

**When is each used?**

Flow auto-detects the right mode from the connector address:
- An IP address (e.g. `"10.0.0.1:3000"`) → `Static`
- A hostname (e.g. `"backend.internal:443"`) → `Dns`

You can override the auto-detection by writing `discovery` explicitly in the `load-balance` block. The most common reason to do this is `discovery "Static"` with a hostname when you want a single stable connection target resolved once at startup (e.g. a well-known external host like `httpbin.org` whose IP rarely changes). Only the first A record is used — if you need resiliency across multiple IPs or automatic failover when an address changes, use `discovery "Dns"` instead.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        discovery "Static"      // force static even though the address is a hostname
    }
    "httpbin.org:443" tls-sni="httpbin.org" proto="h2-or-h1"
}
```

For DNS discovery, use a hostname connector. `discovery "Dns"` is auto-detected from the hostname — only set it explicitly when you also want to tune `dns-interval-secs`:

```kdl
// Plaintext HTTP — common for internal Kubernetes services
connectors {
    load-balance { selection "RoundRobin" }
    "orders-svc.default.svc.cluster.local:8080"
}

// TLS — external service or when the backend requires TLS
connectors {
    load-balance {
        selection "RoundRobin"
        discovery "Dns"         // explicit because we're setting dns-interval-secs
    }
    "catalog.internal:443" proto="h2-or-h1" dns-interval-secs=30
    // tls-sni defaults to the hostname ("catalog.internal") — only set it
    // explicitly when you need a different SNI value.
}
```

**System DNS resolver** — see [DNS Resolution](dns.md) for the full picture. Flow uses `/etc/resolv.conf` by default (correct for Kubernetes, Docker, and bare-metal). To override, add a `dns-resolver` block to `system`:

```kdl
system {
    dns-resolver {
        nameserver "10.96.0.10:53"        // Kubernetes CoreDNS
        nameserver "8.8.8.8:53"           // fallback
        search-domain "svc.cluster.local"
        search-domain "cluster.local"
        ndots 5
        timeout-secs 5
        attempts 2
        ip-version "dual"
        empty-answer-threshold 3
    }
}
```

| Setting | Repeats | Default | Meaning |
|---|---|---|---|
| `nameserver` | yes | *(system)* | DNS server to query, as `IP:port`. Replaces the system resolver entirely when set. Provide at least two for redundancy. |
| `search-domain` | yes | *(system)* | Search domain appended when a bare hostname does not resolve. Equivalent to `search` in `/etc/resolv.conf`. |
| `ndots` | no | *(`resolv.conf`, else 1)* | Dots in a name below which search domains are tried first before a global lookup. Mirrors the `ndots` option in `/etc/resolv.conf`. Kubernetes clusters conventionally use `5`. |
| `timeout-secs` | no | *(`resolv.conf`, else 5)* | Per-query timeout in seconds before the resolver retries or fails. |
| `attempts` | no | *(`resolv.conf`, else 2)* | Number of retry attempts per DNS query before returning a failure. |
| `ip-version` | no | `"dual"` | `"ipv4"` (A records only), `"ipv6"` (AAAA only), or `"dual"` (both). Use `"ipv4"` when backends publish AAAA records Flow cannot reach. Global to all hostname connectors; a name with no records of the selected family counts as an empty answer. See [DNS Resolution](dns.md#which-record-types-are-used). |
| `empty-answer-threshold` | no | `3` | Consecutive NXDOMAIN / empty answers before the backend pool is cleared. Integer ≥ 1. |

`search-domain`, `ndots`, `timeout-secs`, and `attempts` apply with or without `nameserver`. Without `nameserver`, Flow starts from `/etc/resolv.conf` and overrides only the settings you specify. A `search-domain` list replaces the `search` list from `resolv.conf`.

The connector-level `dns-interval-secs` attribute controls how often Flow re-resolves that specific hostname (default `30` seconds). Set it on the connector line, not in `dns-resolver`:

```kdl
"catalog.internal:443" proto="h2-or-h1" dns-interval-secs=30
```

| Connector attribute | Default | Meaning |
|---|---|---|
| `dns-interval-secs` | `30` | Seconds between DNS re-resolution attempts for this hostname. Only applies to `discovery "Dns"` connectors; ignored on IP-address connectors. Lower values pick up DNS changes faster at the cost of more resolver queries. |

**Startup behavior**: Flow resolves every DNS connector hostname at startup before opening any listener. If a hostname fails to resolve at startup, Flow logs a warning and starts with an empty backend pool for that connector — requests to routes backed by that connector receive `502` until the next re-resolution succeeds. Flow does not refuse to start on a DNS failure; a transient resolver blip does not prevent the process from coming up.

**Re-resolution**: background re-resolution runs on the `dns-interval-secs` timer. If a re-resolution fails with a transient error (timeout, SERVFAIL), the existing pool is kept — the last known-good addresses continue serving traffic. If the name returns no addresses (NXDOMAIN or an empty answer), the pool is kept until `empty-answer-threshold` (default 3) consecutive occurrences, then cleared, after which routes return `502`. Flow logs a warning on each failure. A successful re-resolution replaces the pool atomically. See [DNS Resolution](dns.md#when-dns-fails).

**Hostname format**: use `host:port`, not a bare hostname. Flow does not infer a default port from the scheme.

### Health checking

Flow can periodically probe each backend and remove unhealthy ones from the pool.

| `health-check` | Behavior |
|---|---|
| `"None"` | No probing. All backends receive traffic regardless of state. This is the default. |
| `"Active"` | Periodic HTTP GET probe. Backends are removed after `unhealthy-threshold` consecutive failures and restored after `healthy-threshold` consecutive successes. |

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        health-check "Active"
        health-check-path "/healthz"       // path to probe — default /healthz
        health-check-interval-secs 10      // seconds between probes — default 10
        health-check-timeout-secs 2        // probe timeout — default 2
        unhealthy-threshold 3              // consecutive failures to mark down — default 3
        healthy-threshold 2                // consecutive successes to restore — default 2
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

All fields except `health-check "Active"` are optional; the defaults shown apply. Health checking works with both static IP and DNS-discovered connectors.

| Field | Default | Meaning |
|---|---|---|
| `health-check-path` | `"/healthz"` | HTTP path Flow sends the GET probe to. Change this if your backend exposes a health endpoint at a different path. |
| `health-check-interval-secs` | `10` | Seconds between probes per backend. Lower values detect failures faster but add more background traffic. |
| `health-check-timeout-secs` | `2` | How long Flow waits for a probe response before counting it as a failure. Should be less than `health-check-interval-secs`. |
| `unhealthy-threshold` | `3` | Consecutive probe failures before a backend is removed from the pool. Higher values avoid removing a backend on a transient blip. |
| `healthy-threshold` | `2` | Consecutive probe successes before a removed backend is returned to the pool. Prevents flapping on a backend that is still unstable. |

**Fail-open**: when all backends are marked unhealthy, Flow routes requests anyway rather than returning errors immediately. This prevents a misconfigured probe from taking a service offline.

**Active vs passive health checking**: `health-check "Active"` is *active* — Flow sends synthetic probes independent of real traffic, catching backends that are completely down before any client request hits them. It does not catch backends that are alive and pass `/healthz` but are failing real requests (overloaded, partially broken). That is *passive* failure detection, handled by circuit breaking. See [Design Decisions → How Should I Handle Backend Failures?](design-decisions.md#how-should-i-handle-backend-failures) for a full comparison.

### Circuit breaking

Circuit breaking lives inside `load-balance` rather than `path-control` because it operates at the **backend selection layer**, not the request pipeline layer. Policies in `path-control` run after a route is matched but before a backend is chosen — they don't know which specific backend will serve the request. Circuit breaking needs to track the outcome *per backend* (which backend returned a 5xx, which backend's connection failed) and gate future requests to that specific backend. That coupling to backend identity is why it sits alongside `selection`, `health-check`, and `discovery`, not alongside `ip-restriction` or `rate-limit`.

Circuit breaking observes real request outcomes on live traffic. When the error rate (HTTP 5xx responses and connection failures) exceeds a threshold in a time window, the per-backend circuit opens and requests fail fast. After a cooldown the circuit becomes half-open — a small number of probe requests are allowed through; success closes the circuit, failure reopens it.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        circuit-breaker {
            error-threshold-pct 50   // trip at 50% errors — default
            window-secs 60           // 60-second window — default
            min-requests 5           // need 5 requests before tripping — default
            cooldown-secs 30         // 30s open before trying half-open — default
            probe-requests 1         // 1 probe request per half-open cycle — default
            window-strategy "SlidingWindow"  // see Window strategy below — default
        }
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

All fields are optional; the defaults above apply.

| Field | Default | Meaning |
|---|---|---|
| `error-threshold-pct` | `50` | Integer 1–100. The circuit opens when this percentage of requests in the window return a 5xx response. Lower values trip faster; higher values tolerate more errors before opening. |
| `window-secs` | `60` | How far back (in seconds) to look when calculating the error rate. Shorter windows react faster to sudden failures; longer windows smooth over brief spikes. |
| `min-requests` | `5` | Minimum number of requests in the window before the circuit can trip. Prevents tripping on the very first error at startup or after a long idle period. |
| `cooldown-secs` | `30` | How long (in seconds) the circuit stays open before moving to half-open and allowing probe requests. Gives the backend time to recover before traffic resumes. |
| `probe-requests` | `1` | Number of requests allowed through in half-open state. A successful probe closes the circuit; any failure re-opens it and resets the cooldown. Higher values give more confidence before closing but delay recovery. |
| `window-strategy` | `"SlidingWindow"` | How outcomes are tracked — `"SlidingWindow"` (exact, default) or `"TimeBuckets"` (fixed-memory, approximate). See [Window strategy](#window-strategy) below. |
| `bucket-count` | `10` | Number of time buckets. Only valid with `window-strategy "TimeBuckets"`. Must be ≥ 2. Slot width = `window-secs / bucket-count` seconds. |

You can combine `circuit-breaker` with `health-check "Active"` — they catch complementary failure modes. Circuit breaking catches backends that pass health probes but fail real requests; active health checking catches backends that are completely down.

**Fail-open**: the same policy as active health checking — if all backends on a route are circuit-open, Flow routes requests anyway rather than returning errors to every client.

#### Window strategy

Two strategies control how request outcomes are tracked within the window:

| Strategy | How it works | Best for |
|---|---|---|
| `"SlidingWindow"` (default) | Exact per-request tracking. Every outcome is stored with a timestamp; entries outside the window are discarded on each new request. | Most routes. Start here. Memory grows with traffic — at 1,000 req/s with a 60 s window, roughly 1 MB per backend. |
| `"TimeBuckets"` | Divides the window into `bucket-count` fixed-width time slots. Each slot holds aggregate success/failure counts. Stale slots reset lazily on next write. Error rate has ±one slot-width of approximation. | Routes with very high request rates (tens of thousands per second per backend), or when you need a fixed, predictable memory footprint regardless of traffic. |

**Rule of thumb**: use `"SlidingWindow"` unless you have a specific reason not to. Switch to `"TimeBuckets"` when the route is handling tens of thousands of requests per second per backend and you want to cap per-backend circuit-breaker memory to a known constant (a few hundred bytes).

**Example — SlidingWindow (default)**

Exact tracking. Suitable for most routes. `window-strategy` can be omitted since `"SlidingWindow"` is the default.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        circuit-breaker {
            error-threshold-pct 50
            window-secs 60
            min-requests 5
            cooldown-secs 30
            probe-requests 1
            window-strategy "SlidingWindow"
        }
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

**Example — TimeBuckets**

Fixed-memory approximation. Use for high-traffic routes or when predictable memory is required. `bucket-count` sets the number of time slots; the example gives 10 × 6-second slots for a 60-second window.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        circuit-breaker {
            error-threshold-pct 50
            window-secs 60
            min-requests 5
            cooldown-secs 30
            probe-requests 1
            window-strategy "TimeBuckets"
            bucket-count 10
        }
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

`bucket-count` is only valid when `window-strategy` is `"TimeBuckets"` — setting it with `"SlidingWindow"` is a configuration error. The slot width is `window-secs / bucket-count` seconds; with the example values each slot covers 6 seconds, so the error rate can be up to 6 seconds stale at the window boundary.

### Protocol and TLS

| Attribute | Meaning |
|---|---|
| `proto` | `h1-only`, `h2-only`, or `h2-or-h1`. Omitted means HTTP/1.1. See the strictness note below. |
| `tls-sni` | Enables TLS to the upstream, with this SNI/hostname. **Required for HTTP/2 over TLS.** |
| `allow-h2c` | Opt in to **cleartext HTTP/2**. Requires `proto="h2-only"` and **no** `tls-sni`. |
| `verify-cert` | Verify the upstream's certificate chain. Default `true`. |
| `verify-hostname` | Verify the hostname against the cert. Default `true`. Useful to disable when the cert has an IP SAN. |
| `ca-cert-path` | PEM file used as the *exclusive* CA trust store for this connector only. |

**How TLS and HTTP/2 interact:**

| You want | Set |
|---|---|
| HTTP/1.1, cleartext | *(nothing — the default)* |
| HTTP/1.1 over TLS | `tls-sni="host"` `proto="h1-only"` |
| HTTP/2 over TLS | `tls-sni="host"` `proto="h2-only"` (or `h2-or-h1` to allow fallback) |
| **HTTP/2 cleartext (h2c)** | `proto="h2-only"` `allow-h2c=true` |

`allow-h2c` is mutually exclusive with `tls-sni` and rejects `h2-or-h1` — cleartext has no
ALPN, so there is no negotiation and no HTTP/1.1 fallback. The backend **must** accept
prior-knowledge h2c or the connection fails.

> #### `h2-only` is strict — it will not silently fall back to HTTP/1.1
>
> When you set `proto="h2-only"`, Flow **requires** the backend connection to be HTTP/2.
> If the backend does not negotiate HTTP/2 over TLS (it only speaks HTTP/1.1, is
> misconfigured, or an intermediary strips the `h2` ALPN), Flow returns **502** for that
> request rather than quietly proxying it over HTTP/1.1. This surfaces the mismatch instead
> of hiding it — a connector labelled `h2-only` that is actually talking HTTP/1.1 is almost
> always a configuration error you want to know about.
>
> If HTTP/1.1 is an acceptable outcome for that backend, use **`proto="h2-or-h1"`** instead:
> it advertises both protocols and cleanly uses whichever the backend supports (and upgrades
> to HTTP/2 automatically if the backend gains support later — no config change). Cleartext
> **h2c** is inherently strict the same way: a backend that does not speak prior-knowledge
> h2c fails to connect rather than falling back.

Use h2c to drop upstream TLS crypto while keeping HTTP/2 multiplexing. It is the fastest
configuration Flow offers, but the hop is unencrypted — only use it on trusted internal
networks. See [Performance](performance.md).

> ⚠️ **`verify-cert=false` disables all certificate validation** for that connector,
> including against forged certificates. Development and testing only.

### Tuning

| Attribute | Default | H1 | H2 | Meaning |
|---|---|---|---|---|
| `max-h2-streams` | **100** | — | ✅ | Concurrent HTTP/2 streams per upstream connection. |
| `idle-timeout-secs` | none | ✅ | ✅ | How long an idle upstream connection is kept in the pool. |
| `read-timeout-secs` | none | ✅ | ✅ | Max wait per read from the upstream. **Never set on streaming backends** (SSE, gRPC, WebSocket) — fires on idle intervals, not total duration. |
| `write-timeout-secs` | none | ✅ | ✅ | Max wait per write to the upstream. **Never set on streaming backends** for the same reason. |
| `h2-ping-interval-secs` | none | — | ✅ | HTTP/2 keepalive PING interval. Use instead of `read-timeout-secs` on long-lived HTTP/2 connections. |

`read-timeout-secs` and `write-timeout-secs` default to no timeout (unlimited wait). They apply to **both H1 and H2** upstream connections — unlike the downstream timeouts below, which are H1-only.

**`max-h2-streams`** is the one worth understanding. It controls how many requests Flow can
have in flight over a *single* upstream connection. Multiplexing is the main reason to use
HTTP/2 upstream at all: without it, every concurrent request needs its own connection — and
its own TLS handshake — which caps your throughput.

Flow defaults it to **100** on any connector that can negotiate HTTP/2 (`h2-only`,
`h2-or-h1`, or h2c). You do not need to set it. Raise or lower it only if you know your
backend's limit:

- The effective limit is the **smaller** of your value and the backend's advertised
  `SETTINGS_MAX_CONCURRENT_STREAMS`. Setting 1000 against a backend that allows 100 gets you
  100.
- If the backend advertises no limit, your value is the only cap.
- It is ignored on `h1-only` connectors — HTTP/1.1 has no multiplexing. Flow warns at startup
  if you set it there.

Integer attributes reject negative, non-integer, and overflowing values. `0` is rejected
wherever it has no sensible meaning — `max-h2-streams=0` is an error rather than a silent way
to disable HTTP/2 (use `proto="h1-only"` for that).

**Unknown attributes are rejected at load.** A typo'd or misplaced attribute (for example
`max-h2-streams` on a listener line instead of a connector) is a hard error naming the
offending key — not silently ignored. If a setting seems to have no effect, `--validate-configs`
will tell you whether the parser accepted it.

---

## Path control (policies)

Filters that inspect or rewrite requests and responses. They can be attached at the
**service** level (applies to every route) or the **route** level.

```kdl
path-control {
    request-filters {
        filter kind="ip-restriction" deny="10.0.0.0/8"
    }
    upstream-request {
        filter kind="upsert-header" key="x-api-key" value="secret"
    }
    upstream-response {
        filter kind="timing-header"
    }
}
```

See [Policies](policies.md) for every available filter and its options.

---

## Command line

A few settings can also be given on the command line. The interaction is **not** a simple
"CLI wins" rule — switches like `--daemonize` can only be turned *on* from the CLI, and
`--pidfile` / `--upgrade-socket` must *agree* with the config file or Flow refuses to start.
`--license-file` (and its `FLOW_LICENSE_FILE` env-var equivalent) has no config-file form at
all — the license is checked independently of this file, before it's even parsed.

See the [Command-line reference](cli.md) and [Licensing](license.md) for the details.
