# Configuration Reference

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Know what to expect?** See [How Flow Behaves](behavior.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Routing](routing.md) • [Policies](policies.md) • [Performance](performance.md)

Flow is configured with a single [KDL](https://kdl.dev) file, passed with `--config-kdl`.
The file has two top-level blocks: `system` and `services`.

<!-- check-docs: skip -->
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
| `system.metrics` | `listen`, `enabled`, `groups` | — |
| `system.metrics.groups` | `dns`, `bandwidth` | — |
| `system.dns-resolver` | `ndots`, `timeout-secs`, `attempts`, `ip-version`, `empty-answer-threshold` | `nameserver`, `search-domain` |
| `services` | — | named service blocks (service names must be unique) |
| A service | `listeners`, `routes`, `path-control`, `route-cache-dynamic-capacity`, `keepalive-timeout-secs` | — |
| `listeners` | — | listener address lines |
| `routes` | — | `route` blocks (route names must be unique within the service) |
| A route | `name`, `match`, `connectors`, `allowed-methods`, `allowed-protocols`, `path-control`, `cache-response`, `downstream-read-timeout`, `downstream-write-timeout` | — |
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

    // Prometheus metrics endpoint (off unless this block is present).
    // metrics {
    //     listen "127.0.0.1:9090"
    //     groups { dns true  bandwidth true }
    // }
}
```

| Setting | Meaning |
|---|---|
| `threads-per-service` | Worker threads per service. Set this to your **core count**. Hyperthreading does not help a CPU-bound proxy. |
| `daemonize` | Run in the background. If `true`, `pid-file` **must** be set. |
| `pid-file` | Absolute path to the PID file used when daemonizing. |
| `upgrade-socket` | Absolute path to the Unix socket used for zero-downtime upgrades. See [Operations](operations.md). |
| `ca-file` | A CA bundle trusted for **all** outgoing upstream TLS connections. Use when every upstream shares one private CA. For per-upstream CAs, use `ca-cert-path` on the connector instead. |
| `metrics` | A Prometheus metrics endpoint on its own `IP:port` listener (`listen`), with `enabled false` to switch it off and `groups { dns true  bandwidth true }` for optional series. Off unless the block is present. Plain HTTP with no authentication: bind it to loopback or a trusted network. See [Metrics](metrics.md). |
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

An `access-log` block may appear inside `logging`. Flow accepts it without error and does not
read anything inside it. The name is reserved so configs that include it now will not need to
be changed later.

> **Planned:** per-request access logging. See [Roadmap](roadmap.md#observability).

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

### `metrics`

Prometheus endpoint. Off unless the block is present. Full reference, including every rule and error: [Metrics](metrics.md#turning-it-on).

```kdl
system {
    metrics {
        listen "127.0.0.1:9090"     // required: IP:port, not shared with any service listener
        enabled true                // optional, default true; false = validated but nothing starts
        groups {                    // optional; each group defaults to false
            dns true                // flow_dns_* per-hostname DNS discovery series
            bandwidth true          // flow_bandwidth_bytes_total per route
        }
    }
}
```

| Key | Default | Values |
|---|---|---|
| `listen` | *(required)* | `"IP:port"`, IPv4 or IPv6, port not `0` |
| `enabled` | `true` | `true` / `false` |
| `groups.dns` | `false` | `true` / `false` |
| `groups.bandwidth` | `false` | `true` / `false` |

---

## `services` block

Each named child of `services` is an independent proxy service with its own listeners and
routes.

```kdl
services {
    Example1 {
        route-cache-dynamic-capacity 10000
        keepalive-timeout-secs 60

        listeners {
            ...
        }
        routes {
            ...
        }
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

### Listener addresses must be unique and bindable

- **No two listeners may share an address**, whether in one service or in different services.
  Two TCP addresses clash when they have the same port and the same IP, or when either one is a
  wildcard (`0.0.0.0` or `::`) on that port. A clash is rejected when the config loads, and the
  error names both services and the line of the first listener:

  ```
  listener 127.0.0.1:8080 of service "B" conflicts with listener 127.0.0.1:8080 of service "A"
  (line 4); two listeners cannot share an address
  ```

  The same port on different specific IP addresses (for example `127.0.0.1:8080` and
  `10.0.0.1:8080`) does not clash.
- **Every listener must be bindable when Flow starts.** Before starting, Flow tries to bind each
  listener address. If any fails (the port
  is in use, the IP address does not exist on the host, or the port needs privileges), Flow does not
  start and lists every failure:

  ```
  cannot bind configured listeners:
    service "A" cannot listen on 127.0.0.1:19403: Address already in use (os error 48)
    service "B" cannot listen on 192.0.2.1:19405: Can't assign requested address (os error 49)
  ```

  `--validate-configs` checks the clash rule but does not bind anything, so it cannot detect a
  busy port or a missing address. The startup bind check is skipped during a graceful `--upgrade`,
  when the previous process still holds the ports.

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
        routes {
            /* … */
        }
    }
    Public {
        listeners {
            "0.0.0.0:8443" cert-path="…" key-path="…" offer-h2=true
        }
        routes {
            /* … */
        }
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

Each `route` has a `name`, a `match` block, the methods and protocols it accepts, and the
connectors it forwards to.

```kdl
routes {
    route {
        name "echo"                     // required, unique within the service
        match {
            path-prefix "/echo"
        }

        allowed-methods "GET" "PUT" "PATCH" "DELETE" "HEAD" "OPTIONS"
        allowed-protocols "http" "https" "h1" "h2"

        connectors {
            ...
        }
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

### `name` (required)

Every route must have a `name`. It identifies the route, and a missing or invalid name stops
Flow from starting.

```kdl
route {
    name "orders-read"
    match {
        path-prefix "/orders"
    }
    ...
}
```

| Rule | Detail |
|---|---|
| Characters | Lowercase letters, digits, `_`, `.` and `-` |
| First character | A lowercase letter or a digit |
| Length | 1 to 63 characters |
| Uniqueness | Unique **within its service**. Two routes in one service with the same name are rejected, and the error names the line of the first one. The same name may be used in different services. |
| Reserved | `unmatched` |

Service names must also be unique: two services with the same name are rejected at load.

Example error:

```
route name "orders" is already used by another route in this service
(first defined at line 8); route names must be unique within a service
```

Choose names that describe the route's purpose (`orders-read`, `checkout`, `static-assets`).
Renaming a route later is a config change like any other.

### `allowed-methods` (required)

Every route must list the HTTP methods it accepts. A request with a method that isn't
listed does not match the route (and returns **502** if nothing else matches). Method names
are case-insensitive; non-standard methods like `PURGE` work.

### `allowed-protocols` (required)

Every route must list at least one of these. Each request falls into **exactly one** of them:

| Value | The request is… |
|---|---|
| `h2` | HTTP/2, over TLS or in cleartext (h2c) |
| `https` | HTTP/1.x over TLS |
| `h1` | plain (cleartext) HTTP/1.1 |
| `http` | plain (cleartext) HTTP other than 1.1, such as HTTP/1.0 |
| `grpc` | a **gRPC** call from a cleartext HTTP/2 client (declared by you, see below) |
| `grpcs` | a **gRPC** call from a TLS HTTP/2 client (declared by you, see below) |

Because the values do not overlap, `http` alone does **not** admit ordinary plain HTTP/1.1
requests; list `h1` for those.

| To accept… | List |
|---|---|
| Plain HTTP/1.1 requests | `"h1"` |
| HTTP/1.1 over TLS | `"https"` |
| HTTP/2 (TLS or cleartext) | `"h2"` |
| Every kind of request | `"http" "https" "h1" "h2"` |

A request whose kind is not listed is rejected: with **426 Upgrade Required** when it arrived
over plain HTTP and the route lists `https` or `h2` (TLS is the fix), otherwise with **400**.

#### gRPC routes: `grpc` and `grpcs`

Flow does not detect gRPC from the traffic. You declare a gRPC route by listing `grpc`, `grpcs`, or both:

| You write | The route accepts |
|---|---|
| `allowed-protocols "grpc"` | gRPC from clients on a **cleartext** HTTP/2 connection (h2c) |
| `allowed-protocols "grpcs"` | gRPC from clients on a **TLS** HTTP/2 connection |
| `allowed-protocols "grpc" "grpcs"` | either |

- A gRPC route implies `POST`, so `allowed-methods` can be left out. If you write it, it must be exactly `"POST"`.
- A gRPC route lists only `grpc` and/or `grpcs`; do not mix them with `http`, `https`, `h1` or `h2`.
- Every connector on a gRPC route must be HTTP/2 only: `proto="grpc"`, `proto="grpcs"` or `proto="h2-only"`.
- The client's transport is enforced. Plain HTTP/1.1 gets **400**; a cleartext client on a `grpcs`-only route gets
  **426**; a TLS client on a `grpc`-only route gets **400**.
- The route says how **clients** reach Flow; the connector's `proto="grpc"` or `proto="grpcs"` says how Flow reaches the
  **backend**. They are independent, so TLS in and cleartext out is `allowed-protocols "grpcs"` with `proto="grpc"`.

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

Controls the downstream H1 read timeout for this route. The default is **60 seconds** — correct for normal REST routes and for SSE responses, but too short for WebSocket, where the connection is read from the client after the upgrade and a silent connection is closed.

| Value | Meaning |
|---|---|
| absent | Default (60 s) |
| `"None"` | Disabled — no read timeout; required for WebSocket |
| positive integer | Explicit cap in seconds (e.g. `3600`) |

```kdl
downstream-read-timeout "None"   // WebSocket — unbounded
downstream-read-timeout 3600     // generous-but-bounded (1 hour)
```

`0` is rejected. No-op on HTTP/2 downstream connections. See [Non-REST Protocols](non-rest-protocols.md)
for full SSE and WebSocket configuration examples.

### `downstream-write-timeout` (optional)

Controls the downstream H1 write timeout for this route. There is **no write timeout by default** — a stalled client can hold the connection open indefinitely. Set a positive integer to cap how long Flow waits for the client to accept a write.

| Value | Meaning |
|---|---|
| absent | No write timeout (the default) |
| positive integer | Explicit cap in seconds (e.g. `30`) |

```kdl
downstream-write-timeout 30    // drop clients that stall for > 30 s on a write
```

`0` is rejected. No-op on HTTP/2 downstream connections. See [Non-REST Protocols](non-rest-protocols.md)
for streaming protocol configuration examples.

### Choosing timeouts by traffic type

Timeouts guard against stuck connections, but a timeout that is right for a request and response is wrong for a stream that is quiet between messages. Match the settings to the traffic on the route.

| Traffic | `downstream-read-timeout` (HTTP/1.1 clients) | Upstream `read-timeout-secs` and `write-timeout-secs` | `h2-ping-interval-secs` | HTTP/2 |
|---|---|---|---|---|
| REST (request and response) | Default (60 s) | Optional, for example 30 | Not needed | Optional |
| SSE | Default; no setting needed | Never set | 30 on an HTTP/2 upstream | Optional |
| WebSocket | `"None"` | Never set | Not applicable (HTTP/1.1 only) | Not supported |
| gRPC, unary calls | Not applicable (HTTP/2 clients) | Optional, for example 10 | Optional | `offer-h2=true` on the listener; `allowed-protocols "grpc"` or `"grpcs"` on the route; `proto="grpc"` (cleartext) or `proto="grpcs"` with `tls-sni` (TLS) on the connector |
| gRPC, streaming calls | Not applicable | Never set | 30 | Same as unary |

Two more settings apply to client connections. `downstream-write-timeout` reclaims connections from HTTP/1.1 clients that stop reading, and is off by default. `max-h2-streams` needs no setting, because it defaults to 100.

**Why the upstream timeouts differ.** `read-timeout-secs` and `write-timeout-secs` fire when a single read or write waits longer than the limit, so they are idle timers, not limits on how long a call may last. They suit a request and response, where silence means a hung backend, and they cut a stream that is merely quiet. For a stream, use `h2-ping-interval-secs` on HTTP/2 upstreams: Flow probes the connection and closes it only if the peer stops answering. Flow has no limit on the total duration of an HTTP/2 stream; the client's own deadline bounds it.

**REST.** A backend that does not answer within 30 seconds fails the request with a 502.

```kdl
services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET" "POST"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080" read-timeout-secs=30 write-timeout-secs=30
                }
            }
        }
    }
}
```

**SSE.** No upstream timeouts, and an optional write timeout to drop clients that stop reading. With an HTTP/2 upstream, add a ping interval:

```kdl
services {
    Events {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "events"
                match {
                    path-prefix "/events"
                }
                downstream-write-timeout 30
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.6:8080" proto="h2-only" allow-h2c=true h2-ping-interval-secs=30
                }
            }
        }
    }
}
```

For an HTTP/1.1 upstream, drop `proto`, `allow-h2c`, and the ping interval; there is nothing to tune upstream.

**WebSocket.** A WebSocket is read from the client after the upgrade, and the default 60-second read timeout closes a silent connection, so disable it. The upstream connector sets no timeouts.

```kdl
services {
    Realtime {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "ws"
                match {
                    path-prefix "/ws"
                }
                downstream-read-timeout "None"
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.7:8080"
                }
            }
        }
    }
}
```

**gRPC.** The listener accepts cleartext HTTP/2, and the connector speaks HTTP/2 to the backend. Timeouts belong to the connector, so give each call type its own route: the unary method has a read timeout, and the streaming method has a ping interval and no read timeout. Both routes can point to the same backend and share one connection, and each keeps its own settings.

```kdl
services {
    Grpc {
        listeners {
            "0.0.0.0:8080" offer-h2=true
        }
        routes {
            route {
                name "orders-get"
                match {
                    path-prefix "/shop.Orders/GetOrder"
                }
                allowed-protocols "grpc"
                connectors {
                    "10.0.0.8:50051" proto="grpc" read-timeout-secs=10 write-timeout-secs=10
                }
            }
            route {
                name "orders-watch"
                match {
                    path-prefix "/shop.Orders/WatchOrders"
                }
                allowed-protocols "grpc"
                connectors {
                    "10.0.0.8:50051" proto="grpc" h2-ping-interval-secs=30
                }
            }
        }
    }
}
```

For TLS from gRPC clients, give the listener a certificate and key (`cert-path`, `key-path`), keep `offer-h2=true`, and use `allowed-protocols "grpcs"`. For TLS to the backend, use `proto="grpcs"` with `tls-sni` on the connector. See [Non-REST Protocols](non-rest-protocols.md) for the SSE, WebSocket, and gRPC details.

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
- Session affinity (e.g. cache locality, or sticky sessions on backends that keep state in memory) → `Ketama`
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
    load-balance {
        selection "RoundRobin"
    }
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

`health-check` in the `load-balance` block controls backend probing. `"None"` (the default) sends no probes; `"Active"` sends a periodic HTTP GET to each backend. Health checking works with static and DNS-discovered connectors.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        health-check "Active"
        health-check-path "/healthz"       // default
        health-check-interval-secs 10      // default
        health-check-timeout-secs 2        // default
        unhealthy-threshold 3              // default
        healthy-threshold 2                // default
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

All fields except `health-check "Active"` are optional. For what each field does, how probing works, and when to combine it with circuit breaking, see [Health checking](policies/health-checking.md).

### Circuit breaking

`circuit-breaker` is a block inside `load-balance` that trips a backend when its error rate on real traffic exceeds a threshold.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        circuit-breaker {
            error-threshold-pct 50           // default
            window-secs 60                   // default
            min-requests 5                   // default
            cooldown-secs 30                 // default
            probe-requests 1                 // default
            window-strategy "SlidingWindow"  // default; or "TimeBuckets"
        }
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

All fields are optional. `bucket-count` (default `10`, at least `2`) is valid only with `window-strategy "TimeBuckets"`. For what each field does, the state machine, the window strategies, and tuning, see [Circuit breaker](policies/circuit-breaker.md).

### Protocol and TLS

| Attribute | Meaning |
|---|---|
| `proto` | `h1-only`, `h2-only`, `h2-or-h1`, `grpc`, or `grpcs`. Omitted means HTTP/1.1. `grpc` and `grpcs` declare a gRPC backend: see below. See the strictness note below. |
| `tls-sni` | Enables TLS to the upstream, with this SNI/hostname. **Required for HTTP/2 over TLS.** |
| `allow-h2c` | Opt in to **cleartext HTTP/2**. Requires `proto="h2-only"` and **no** `tls-sni`. `proto="grpc"` implies it. |
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

> #### Declaring a gRPC backend: `proto="grpc"` and `proto="grpcs"`
>
> Flow does not detect gRPC from the traffic. You declare it on the connector, and you choose the transport:
>
> | You write | It means | Same as |
> |---|---|---|
> | `proto="grpc"` | gRPC over cleartext HTTP/2 | `proto="h2-only" allow-h2c=true` |
> | `proto="grpcs"` and `tls-sni="<name>"` | gRPC over TLS | `proto="h2-only" tls-sni="<name>"` |
>
> Both are strict, like `h2-only`: a backend that does not speak HTTP/2 fails with a 502. `proto="grpc"` cannot be combined with
> `tls-sni`, and `proto="grpcs"` requires it and cannot be combined with `allow-h2c`. The longer spellings stay valid.

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
| `read-timeout-secs` | none | ✅ | ✅ | Max wait per read from the upstream. **Never set on streaming backends** (SSE, streaming gRPC methods, WebSocket) — fires on idle intervals, not total duration. |
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
