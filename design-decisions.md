# Design Decisions

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Understanding limitations?** See [Limitations](limitations.md)
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

- **Encryption cost**: ~−29% throughput (TLS vs cleartext)
- **Multiplexing benefit**: ~+51% throughput (HTTP/2 vs HTTP/1.1)
- **Full end-to-end multiplexing** (h2c both hops) delivers the most throughput

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

### Consistent Hashing (FVN or Ketama)

- **How it works**: Hash request properties (usually source IP or URI path) to consistently route to the same backend
- **Best for**: Sticky sessions, cache locality, or when backend state matters (session affinity)
- **Which hash function**: 
  - **FVN** (FowlerNollVo): General purpose, faster
  - **Ketama**: Used by memcached, if you're already using Ketama elsewhere
- **What gets hashed**: Configurable via `RequestSelector` (source IP + path, path only, etc.)
- **Configuration**: `selection "FVNHash"` or `selection "KetamaHashing"`, choose selector

### Considerations

- **Uneven backend capacity**: No built-in weighting. If backends have different capacity, use round-robin or configure load-test to verify distribution.
- **Resilience**: If a backend fails, consistent hashing still routes to it (requires health checking, not yet in Flow). For now, all backends must be healthy.

See [Configuration Reference](configuration.md) for syntax.

---

## Which Policies Should I Use?

Policies are filters that run on requests (rate limiting, IP blocking) or responses (header rewriting, timing injection). Apply them per-route.

### Rate Limiting

Prevent traffic spikes and enforce fair use.

**Types**:
1. **Global rate limit** — one shared bucket for all requests to a route
2. **Per-source-IP rate limit** — separate bucket per client IP
3. **Per-URI-path rate limit** — separate bucket per request path

**Example scenario**:
- `rate-limit-single-uri-group 100/sec` → max 100 req/sec to this route (shared bucket)
- `rate-limit-multi-source-ip 50/sec` → max 50 req/sec per unique client IP

See [Rate Limiting Policy](policies/rate-limiting.md) for detailed configuration.

### IP Blocking (CIDR Ranges)

Block traffic from specific IP ranges.

**Example**: Block all requests from `192.168.1.0/24`

```kdl
path-control {
  block-cidr-range {
    ranges "192.168.1.0/24" "10.0.0.0/8"
  }
}
```

See [Block CIDR Range Policy](policies/block-cidr-range.md).

### Header Rewriting

Add, remove, or modify headers in requests or responses.

**Example**: Add `X-Forwarded-By: Flow` to upstream requests

```kdl
upstream-request {
  upsert-header {
    key "X-Forwarded-By"
    value "Flow"
  }
}
```

See [Upsert Header Policy](policies/upsert-header.md) and [Remove Header Policy](policies/remove-header-key-regex.md).

### Timing Headers

Inject request/response latency measurements (diagnostic).

```kdl
upstream-response {
  timing-header {}
}
```

Adds `X-Upstream-Duration-Us` and `X-Flow-Duration-Us` headers with microsecond-precision timings.

See [Timing Header Policy](policies/timing-header.md).

---

## How Should I Structure My Routes?

Routes match incoming requests and select a backend. Matching is based on path, headers, or SNI. Flow scores all matches and picks the most specific one.

### Matching Strategies

**Path Prefix** — match if request path starts with a prefix

```kdl
match { path-prefix "/api/v1/" }
```

Simple, fast, works for most APIs.

**Path Regex** — match if request path matches a regex

```kdl
match { path-regex "^/users/[0-9]+$" }
```

More expressive, slightly slower.

**Header** — match if request header equals a value

```kdl
match { header "X-API-Key" "secret123" }
```

Useful for tenant routing, feature flags, internal routing.

**SNI (Server Name Indication)** — match if TLS SNI matches a value

```kdl
match { sni "api.example.com" }
```

Virtual hosting at the TLS layer.

**SNI Suffix** — match if SNI ends with a suffix

```kdl
match { sni-suffix ".internal.example.com" }
```

Wildcard SNI matching.

### Combining Matches (AND, OR)

Routes can combine matchers using AND and OR logic.

**AND** — all matchers must match

```kdl
match { all {
  path-prefix "/api/"
  header "X-Tenant" "acme"
} }
```

**OR** — any matcher can match

```kdl
match { any {
  sni "v1.api.example.com"
  sni "v2.api.example.com"
} }
```

### Scoring and Specificity

More specific routes win. Flow scores each match and picks the highest score:

- Longer prefix wins
- Regex match scored by length of matched text
- SNI exact match scores highest (very specific)
- Header match adds a fixed bonus

See [Routing](routing.md) for detailed scoring rules.

### Recommendations

- **Simple APIs**: Use path prefix matching
- **Multi-tenant**: Add header matching (e.g., tenant ID)
- **Virtual hosting**: Use SNI matching
- **Complex routing**: Combine with AND/OR logic, let scoring handle precedence

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
- **Header size**: ~1 MB per connection (not currently tunable in Flow)
- **CPU-bound**: Single core per service thread saturates at a rate depending on protocol/TLS
- **Memory**: Grows with concurrent connections (~40 MB typical)
- **Regex in SNI**: Not supported — only exact match and suffix match

See [Architectural Constraints](architecture-constraints.md) for details and workarounds.

### Known Behaviors

- **All backends fail**: Returns 502 (no graceful degradation)
- **h2-only backend negotiates HTTP/1.1**: Returns 502 (strict protocol enforcement)
- **Rate limiting scope**: Per-Flow instance, not shared across multiple Flow processes

See [Known Behaviors](known-behaviors.md) for more.

### Limitations

Some features are not yet supported:
- WebSocket proxying
- gRPC (implemented but not validated)
- Upstream health checking

See [Not Yet Supported](not-yet-supported.md).

---

## Future Decision Topics

We plan to add guidance on:
- Health checking strategy
- Caching strategy (if implemented)
- Logging strategy
- Request/response buffering
- Graceful shutdown and upgrades
