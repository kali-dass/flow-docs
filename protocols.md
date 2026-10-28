# Protocols

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Know what to expect?** See [How Flow Behaves](behavior.md)

## Supported Protocols

Flow supports HTTP/1.1, HTTP/2, and cleartext HTTP/2 (h2c) on both client-facing and upstream sides. TLS can be used optionally on either hop.

| Protocol | Client→Flow | Flow→Upstream | Encryption | Notes |
|---|---|---|---|---|
| HTTP/1.1 | ✅ | ✅ | Optional | Standard protocol, one request per connection |
| HTTP/2 | ✅ | ✅ | Via TLS + ALPN | Requires TLS for ALPN negotiation |
| HTTP/2 Cleartext (h2c) | ✅ | ✅ | None | Prior-knowledge HTTP/2, for trusted networks |
| TLS | ✅ | ✅ | Yes | Cipher/version/certificate configurable per upstream |

## Protocol Details

### HTTP/1.1

The classic protocol: one HTTP request per connection. Safe, universally supported, straightforward to debug. Without multiplexing, achieving high throughput requires many concurrent connections, each with its own memory footprint and TCP overhead.

**Throughput baseline**: ~14,300 req/s (cleartext) or ~14,300 req/s (TLS).

### HTTP/2 over TLS

HTTP/2 allows multiple streams to share one connection (multiplexing). ALPN (Application Layer Protocol Negotiation) is used during the TLS handshake to agree on protocol version. **Requires TLS** — browsers and most clients enforce this.

Multiplexing recovers much of the per-connection cost: ~1.5× throughput gain over HTTP/1.1 depending on configuration.

**Throughput**: ~20,000 req/s (TLS, both hops) or ~17,000 req/s (TLS client, TLS HTTP/1.1 backend).

### HTTP/2 Cleartext (h2c)

HTTP/2 without encryption. Uses "prior knowledge" — no ALPN negotiation, no TLS handshake. The client and server must both be configured in advance to speak HTTP/2 cleartext.

**Best for**: Trusted internal networks where encryption overhead is unnecessary and maximum throughput matters. Example: microservices in a private datacenter or Kubernetes cluster, where TLS is handled at the perimeter.

Multiplexing removes per-connection churn entirely: ~2× throughput gain over HTTP/1.1.

**Throughput baseline**: ~28,000 req/s (both hops) or ~22,700 req/s (client h2c, backend HTTP/1.1).

### TLS Termination

Flow can **terminate TLS** from incoming clients and/or **originate TLS** to upstream backends. Each upstream can have its own certificate authority, cipher suite, and protocol version — no one-size-fits-all TLS config.

- **From clients**: clients connect with TLS (HTTPS), Flow decrypts and re-encrypts to upstream.
- **To upstreams**: Flow originates a new TLS connection to each upstream independently.
- **Hybrid**: TLS from clients, cleartext to internal upstreams (common pattern).

### Client address behind a TCP load balancer

Flow decides who the client is from the connection's peer address, from `X-Forwarded-For`, or from a header you name (see
[Real client IP](configuration.md#real-client-ip)). Those work when the thing in front of Flow is an HTTP-aware proxy or
CDN that adds the header.

A load balancer that works at the TCP level, or one that passes TLS through untouched, cannot add an HTTP header. It can
forward the client's address in a **PROXY protocol** header at the start of the connection. Flow does not read that header
today, so behind such a load balancer every connection appears to come from the load balancer, and policies keyed on the
client address ([IP restriction](policies/ip-restriction.md), per-IP [rate limiting](policies/rate-limiting.md)) act on the
wrong address.

Until then, either use a load balancer mode that preserves the client's source address, or put an HTTP-aware proxy in front
of Flow and use `X-Forwarded-For`.

> **Planned:** reading the PROXY protocol header from trusted load balancers. See [Roadmap](roadmap.md#protocols).

### Strict HTTP/2 (`h2-only`)

A connector with `proto="h2-only"` must talk HTTP/2 to the backend. If the backend answers over HTTP/1.1 instead (it does not support HTTP/2, it does not advertise `h2` during the TLS handshake, or something in between strips it), Flow returns **502 Bad Gateway** and does not send the request over HTTP/1.1. Some proxies fall back silently; Flow does not, because a silent fallback hides a misconfiguration and costs you the multiplexing you configured for.

With cleartext HTTP/2 (`allow-h2c=true`) there is no negotiation to fall back from: a backend that does not speak HTTP/2 on that port fails to connect.

If HTTP/1.1 is acceptable for the backend, use `proto="h2-or-h1"`:

```kdl
"backend.example.com:443" proto="h2-or-h1" tls-sni="backend.example.com"
```

To diagnose a 502 from an `h2-only` connector, see [Troubleshooting](operations.md#502s-on-a-connector-with-protoh2-only).

## Mixed Protocols

Flow speaks **different protocols on each hop**:

- Client connects with HTTP/1.1, Flow uses h2c to backend ✅
- Client uses HTTP/2 (over TLS), Flow uses HTTP/1.1 to backend ✅
- Client uses HTTP/2, Flow uses h2c to backend ✅

Protocol mismatch is fine — Flow translates between them. The only constraint: if a backend is h2c, it must accept HTTP/2 cleartext on a pre-agreed port (no ALPN).

## Performance Characteristics

Throughput depends on three factors:

1. **Encryption**: TLS costs throughput compared with cleartext.
2. **Protocol**: HTTP/2 multiplexing gains throughput compared with HTTP/1.1.
3. **Multiplexing scope**: multiplexing on both hops gains the most; multiplexing on the client hop only gains less.

For measured throughput, latency percentiles, memory, and CPU for each combination, see [Performance](performance.md).

## Configuration Examples

See [Configuration Reference](configuration.md) for complete syntax. Here are protocol-specific snippets:

### HTTP/1.1 Upstream

```kdl
connectors {
  "backend.example.com:80"
}
```

Simplest case — plain HTTP/1.1, no encryption.

### HTTP/2 with TLS (ALPN)

```kdl
connectors {
  upstream {
    proto "h2-or-h1"
    tls-sni "backend.example.com"
  }
  "backend.example.com:443"
}
```

`h2-or-h1` (or `h2-only` for strict HTTP/2) triggers ALPN during TLS handshake. Backend negotiates HTTP/2 or falls back to HTTP/1.1.

### HTTP/2 Cleartext (h2c)

```kdl
connectors {
  upstream {
    proto "h2-only"
    allow-h2c true
  }
  "internal-backend.local:8080"
}
```

`allow-h2c true` enables cleartext HTTP/2 (prior knowledge). Requires `proto "h2-only"` and **no** `tls-sni`. Backend must accept HTTP/2 cleartext on this port.

## Next Steps

- **Using WebSocket, gRPC, SSE, or GraphQL?** See [Non-REST Protocols](non-rest-protocols.md).
- **Choosing a protocol?** See [Design Decisions](design-decisions.md) for scenarios and trade-offs.
- **Configuring Flow?** See [Configuration Reference](configuration.md) for complete KDL syntax.
- **Evaluating throughput?** See [Performance](performance.md) for detailed benchmarks and capacity planning.
- **Understanding stream limits?** See [How Flow Behaves](behavior.md#http2-stream-concurrency) for HTTP/2 stream concurrency, and [Performance](performance.md#memory) for memory and CPU limits.
