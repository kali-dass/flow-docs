# Design Decisions

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Know what to expect?** See [How Flow Behaves](behavior.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

Making architectural decisions for your Flow setup? This guide helps you choose the right combination of protocols, load balancing, policies, and routing based on your requirements.

## Which Protocol Should I Use?

Choose based on your network topology and performance requirements.

### Decision Tree

**Is your client over the internet (untrusted network)?**
- **Yes, and backend is also untrusted** → [TLS + HTTP/2 (both hops)](#scenario-tls--http2-both-hops)
- **Yes, but backend is internal (trusted)** → [TLS to client, h2c to backend](#scenario-tls-to-client-h2c-to-backend)
- **No, internal network only** → [HTTP/1.1 or h2c](#scenario-internal-network-only)

### Scenario: TLS + HTTP/2 (both hops)

- **Best for**: Internet-facing gateways proxying to untrusted upstreams or public APIs
- **Encryption**: Full end-to-end TLS
- **Multiplexing**: Yes, both hops
- **Throughput**: ~20,000 req/s on 2 cores
- **Complexity**: Highest — manage certificates on both sides
- **How to configure**: See [Configuration Reference](configuration.md), set `proto "h2-or-h1"` with `tls-sni` on both listener and connector

### Scenario: TLS to Client, h2c to Backend

- **Best for**: Public-facing gateway with internal microservices. Typical enterprise setup.
- **Encryption**: TLS to clients, cleartext internally
- **Multiplexing**: Client hop multiplexed, backend hop multiplexed
- **Throughput**: ~22,700 req/s on 2 cores (best overall)
- **Complexity**: Medium — TLS only on public side, internal backends must support h2c
- **Why this is popular**: Best throughput-to-complexity ratio
- **How to configure**: Listener with TLS, connector with `allow-h2c true` and `proto "h2-only"`
- **Security model**: TLS protects the public-facing hop; internal h2c assumes a trusted network. **Never skip TLS on the client-facing side.** The performance gain from h2c to backend is only valid if client encryption is already in place.

### Scenario: Internal Network Only

- **Best for**: Service-to-service proxying, Kubernetes ingress, internal APIs
- **Encryption**: None (trust your internal network)
- **Multiplexing**: Yes (h2c) or no (HTTP/1.1)
- **Throughput**: ~28,000 req/s (h2c, both hops) or ~18,500 req/s (HTTP/1.1)
- **Complexity**: Lowest — no TLS keys to manage
- **Which protocol**: Choose h2c if backends support it (2× throughput over HTTP/1.1), otherwise HTTP/1.1
- **How to configure**: No `tls-sni` on listener, connector with `allow-h2c true` if using h2c

> **🔐 CRITICAL SECURITY NOTE:** Cleartext HTTP/2 (h2c) should **only** be used on networks you fully control and trust. "Internal" means: isolated network segments, no untrusted users/devices, no cross-tenant traffic. If there is any doubt about network isolation, **use TLS**. Security considerations take precedence over performance gains. A 2× throughput improvement is meaningless if the trade-off is credential exposure or data interception.

### Throughput Trade-offs

See [Performance](performance.md) for detailed numbers. Quick summary:

- **Encryption** costs throughput: cleartext is faster than TLS
- **Multiplexing** adds throughput: HTTP/2 is faster than HTTP/1.1
- **Full end-to-end multiplexing** (h2c on both hops) delivers the most throughput

More details in [Protocols](protocols.md).

---

## Which Load Balancing Strategy Should I Use?

Choose based on your backend topology and traffic pattern.

### Round-Robin

- **How it works**: Distribute requests evenly across backends in a repeating cycle
- **Best for**: Homogeneous backends with equivalent capacity and latency
- **Sticky sessions**: No — each request goes to the next backend
- **Configuration**: `selection "RoundRobin"` (default)

### Random

- **How it works**: Randomly pick a backend for each request
- **Best for**: Rough load distribution when you don't care about perfect balance
- **Sticky sessions**: No
- **Configuration**: `selection "Random"`

### Consistent Hashing (FNV or Ketama)

- **How it works**: Hash request properties (usually source IP or URI path) to consistently route to the same backend
- **Best for**: Sticky sessions, cache locality, or when backend state matters (session affinity)
- **Which hash function**: 
  - **FNV** (Fowler–Noll–Vo): General purpose, faster
  - **Ketama**: Used by memcached, if you're already using Ketama elsewhere
- **What gets hashed**: Configurable via `RequestSelector` (source IP + path, path only, etc.). The "source IP" is the resolved client IP — configure `real-ip-source` in `system` for correct hashing behind a proxy or CDN. See [Configuration](configuration.md#real-client-ip).
- **Configuration**: `selection "FNV"` or `selection "Ketama"`, choose selector

### Considerations

- **Uneven backend capacity**: No built-in weighting. If backends have different capacity, use round-robin or configure load-test to verify distribution.
- **Resilience**: If a backend fails, consistent hashing still routes to it unless health checking is enabled. Enable `health-check "Active"` in the `load-balance` block to automatically remove failed backends and restore them when they recover.

See [Configuration Reference](configuration.md) for syntax.

---

## How Should I Handle Backend Failures?

Flow has two complementary mechanisms. They catch different failure modes and work best together.

- **Active health checking** sends periodic probes to each backend, independent of real traffic. It catches backends that are down or unreachable. See [Health checking](policies/health-checking.md).
- **Circuit breaking** watches real request outcomes and trips a per-backend circuit when the error rate passes a threshold. It catches backends that are alive and pass probes but fail real requests. See [Circuit breaker](policies/circuit-breaker.md).

### Which to use

| Scenario | Use |
|---|---|
| Backend crashes or becomes unreachable | Active health checking |
| Backend is alive but returning errors | Circuit breaking |
| Both | Both together — they cover complementary failure modes |
| No session state, orchestrator handles health | Neither — let Kubernetes remove pods from DNS |

**Fail-open**: both mechanisms use fail-open when all backends are affected. Flow routes requests anyway rather than serving 502 to every client, so a misconfigured probe or threshold cannot take a service offline.

---

## Which Policies Should I Use?

Policies are filters that run on requests (rate limiting, IP restriction, authentication) or responses (header rewriting, timing injection). Apply them per route or per service. See [Policies](policies.md) for where each attaches and in what order they run.

| You want to | Use |
|---|---|
| Verify a bearer, cookie, or query token | [Token authentication](policies/token-auth.md) |
| Cap request rates, or enforce fair use per client | [Rate limiting](policies/rate-limiting.md) |
| Allow or block source IP ranges | [IP restriction](policies/ip-restriction.md) |
| Add or change a header | [Upsert header](policies/upsert-header.md) |
| Strip headers by name | [Remove header](policies/remove-header-key-regex.md) |
| Trace a request across services | [Correlation ID](policies/correlation-id.md) |
| See latency per response | [Timing header](policies/timing-header.md) |
| Answer immediately without calling the backend (maintenance, kill switch) | [Request termination](policies/request-termination.md) |
| Serve repeated requests from memory | [Cache response](policies/cache-response.md) |

---

## How Should I Structure My Routes?

Routes match incoming requests and select a backend. Matching is based on path, headers, or SNI, and Flow picks the most specific match. See [Routing](routing.md) for the matchers, how to combine them with `all` and `any`, and the scoring rules.

| Situation | Match on |
|---|---|
| Simple APIs | `path-prefix` |
| Pattern-based paths | `path-regex` |
| Multi-tenant | `header` (for example a tenant ID), usually together with a path |
| Virtual hosting | `sni` or `sni-suffix` |
| Complex routing | `all` and `any` combinations; let scoring settle precedence |

---

## How Should I Handle TLS Certificates?

TLS is optional and can be configured per-listener and per-upstream.

### Listener-Side TLS (Clients)

Each listener can have TLS enabled with a certificate and private key.

```kdl
listener {
  address "0.0.0.0:443"
  tls {
    cert-path "/path/to/cert.pem"
    key-path "/path/to/key.pem"
  }
}
```

All routes under this listener accept HTTPS from clients.

### Upstream-Side TLS (Backend)

Each connector (upstream) can originate TLS independently.

```kdl
connector {
  upstream {
    tls-sni "backend.example.com"
  }
  "backend.example.com:443"
}
```

Flow initiates TLS to the backend with SNI set to `backend.example.com`.

### Per-Upstream CA (Certificate Authority)

Upstreams can have custom CA certificates for self-signed backends.

```kdl
connector {
  upstream {
    tls-sni "internal-backend.local"
    ca-cert-path "/path/to/internal-ca.pem"
  }
  "internal-backend.local:443"
}
```

### Hybrid: TLS from Clients, Cleartext to Backends

Common pattern — encrypt traffic from the internet, trust internal backends.

**Listener**: Configure with TLS (public-facing)
**Connector**: No `tls-sni` (cleartext to backend)

---

## What Should I Be Aware Of?

Flow has architectural constraints and known behaviors worth understanding when designing your setup.

### Architectural Constraints

- **HTTP/2 stream limit**: Default 100 concurrent streams per connection (tunable via `max-h2-streams`)
- **Header size**: ~1 MB per connection (not currently tunable in Flow; see [How Flow Behaves](behavior.md#header-size))
- **CPU-bound**: Single core per service thread saturates at a rate depending on protocol/TLS
- **Memory**: Grows with concurrent connections (~40 MB typical)
- **Regex in SNI**: Not supported — only exact match and suffix match

See [Performance](performance.md) for CPU and memory, [How Flow Behaves](behavior.md#http2-stream-concurrency) for stream limits, and [Routing](routing.md) for SNI matching.

### Known Behaviors

- **All backends fail**: Returns 502 (no graceful degradation)
- **h2-only backend negotiates HTTP/1.1**: Returns 502 (strict protocol enforcement)
- **Rate limiting scope**: Per-Flow instance, not shared across multiple Flow processes

See [How Flow Behaves](behavior.md) and [Protocols](protocols.md#strict-http2-h2-only) for more.

### Roadmap

For what is available and what is planned, including WebSocket and gRPC status, see the [Roadmap](roadmap.md).

---
