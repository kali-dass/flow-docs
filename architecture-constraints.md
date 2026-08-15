# Architectural Constraints

> **See also:** [Limitations Hub](limitations.md) • [Known Behaviors](known-behaviors.md) • [Not Yet Supported](not-yet-supported.md) • [What Flow Is Not](scope.md)

Flow has inherent design limits. Understanding these helps you size deployments and make architectural decisions.

## HTTP/2 Stream Limit

**Constraint**: Flow limits concurrent HTTP/2 streams per connection.

**Default**: 100 streams per connection (configurable via `max-h2-streams` on each connector)

**Why this matters**: HTTP/2 multiplexes multiple requests over one connection. If you have more than 100 concurrent requests, they queue on the connection until earlier requests complete. This is rarely a problem because:
- The limit is per connection, not global
- Each new connection can have 100 new streams
- Most clients open multiple connections automatically
- 100 streams per connection is more than most backends need

**When to increase**: If you see connection queueing in traces or Flow's request latencies spike under high concurrency, increase `max-h2-streams` on the connector. The backend's `SETTINGS_MAX_CONCURRENT_STREAMS` is the ultimate limit (Flow's limit is ignored if lower).

**Configuration**:

```kdl
connector {
  upstream {
    max-h2-streams 200
  }
  "backend.example.com:443"
}
```

**Performance impact**: Negligible when tuned appropriately.

---

## Header Size Limit

**Constraint**: Flow allows up to ~1 MB of headers per connection.

**Why this matters**: Each request's headers (including cookies) must fit in 1 MB combined. Most APIs don't come close to this, but:
- Large cookies can accumulate
- Custom headers (tracing, auth tokens) add up
- Some old APIs with verbose headers might hit this

**When you'll hit it**: Rare. Almost never in practice unless you have enormous cookies or hundreds of custom headers.

**Workaround**: Compress or trim header data before sending to Flow. If Flow rejects headers, you'll see an error in logs.

**Future**: Not currently tunable in Flow. If needed, we can make it configurable.

---

## CPU Saturation

**Constraint**: Flow is CPU-bound. Single core per service thread saturates at a specific throughput depending on protocol and encryption.

**Measured ceilings** (2 CPU cores, 5-minute sustained runs):
- HTTP/1.1 + TLS (both hops): ~14,300 req/s
- HTTP/1.1 + cleartext: ~18,500 req/s
- HTTP/2 + TLS (both hops): ~20,000 req/s
- h2c + HTTP/1.1 backend: ~22,700 req/s
- h2c (both hops): ~28,000 req/s

**Why this matters**: Once CPU saturates (95%+), throughput stops increasing. Adding more load causes requests to queue and latency to spike. This is normal and expected.

**How to operate past the ceiling**:
- Add more CPU cores (horizontal scaling)
- Run multiple Flow instances on different cores
- Reduce TLS encryption cost by using h2c internally

**Details**: See [Performance](performance.md) for CPU and memory breakdowns at each throughput level.

---

## Memory Footprint

**Constraint**: Memory grows with concurrent connections and buffer sizes.

**Typical usage** (2 CPU cores, under sustained load):
- HTTP/1.1 + TLS: ~35 MB
- HTTP/2 + TLS (both hops): ~43–53 MB
- h2c (both hops): ~32–43 MB

**Why this matters**: Each connection holds buffers for request and response data. More concurrent connections = more memory. Memory is secondary to CPU; CPU saturates first.

**How to reduce memory**:
- Reduce buffer sizes (not easily tunable currently, mostly fixed by Flow's architecture)
- Reduce concurrent connections (client-side connection pooling)
- Use HTTP/1.1 or h2c/HTTP/1.1 (lower memory than TLS+HTTP/2)

**Headroom**: Plan for ~50 MB per Flow instance under moderate load. Larger deployments will scale linearly.

---

## No Regex in SNI Matching

**Constraint**: SNI (TLS hostname) matching supports only exact match and suffix match (wildcard).

**Supported**:
- Exact: `sni "api.example.com"` (matches only `api.example.com`)
- Suffix: `sni-suffix ".example.com"` (matches `api.example.com`, `v2.example.com`, etc.)

**Not supported**:
- Regex: `sni-regex "^(api|admin)\.example\.com$"` ❌

**Why this matters**: If you need complex SNI patterns, you must pre-process at the TCP layer (nginx, HAProxy) or split listeners.

**Workaround**:
- Use header matching instead of SNI for complex logic
- Run separate Flow instances for different SNI patterns (e.g., one for `*.api.example.com`, one for `*.admin.example.com`)
- Use a reverse proxy in front of Flow to handle regex SNI, then forward to Flow

---

## Protocol Matching Strictness (h2-only)

**Constraint**: If you configure a backend with `proto "h2-only"`, Flow refuses HTTP/1.1 responses. Flow returns 502 Bad Gateway if the backend negotiates HTTP/1.1 instead of HTTP/2.

**Why this matters**: This is a design decision to be strict about protocol compliance. Some proxies silently accept HTTP/1.1 fallback from an `h2-only` backend, but Flow treats it as an error because:
- It signals a configuration mismatch (backend not actually h2-only)
- Silent fallback masks production issues
- Strict mode forces you to fix the backend

**When you'll hit it**: If you misconfigure a backend as `h2-only` but it only speaks HTTP/1.1, or if the backend's TLS negotiation fails to select HTTP/2.

**How to fix**:
- Verify the backend supports HTTP/2 and ALPN
- Use `proto "h2-or-h1"` to allow fallback if HTTP/2 is not critical
- Check TLS configuration (cert, cipher suite) on the backend

**Example**:

```kdl
# Strict (502 if HTTP/1.1 negotiated)
upstream { proto "h2-only" }

# Permissive (accept HTTP/1.1 fallback)
upstream { proto "h2-or-h1" }
```

---

## No Cross-Instance Rate Limiting

**Constraint**: Rate limiting is per-Flow instance, in-memory. If you run multiple Flow instances, each has independent rate limit buckets.

**Why this matters**: If you have 2 Flow instances and set a 100 req/sec global rate limit, each instance independently allows 100 req/sec. Total throughput is 200 req/sec, not 100 req/sec.

**Workaround**:
- **Run a single Flow instance**: If you need strict rate limits, run one instance (with multiple threads/cores).
- **Use external rate limiting**: Implement rate limiting outside Flow (Redis-based) — **or wait for distributed rate limiting** which is planned for Flow.
- **Accept per-instance limits**: Rate limit per instance knowing the ceiling multiplies by instance count.

**Roadmap**: Distributed rate limiting is planned for a future release to enable shared rate limit buckets across multiple Flow instances.

---

## Next Steps

- **Sizing a deployment?** See [Performance](performance.md) for detailed capacity numbers
- **Designing around constraints?** See [Design Decisions](design-decisions.md) for architecture guidance
- **Want to know how to operate?** See [Operations](operations.md) for scaling, upgrades, and monitoring
