# Protocols

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)

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

## Mixed Protocols

Flow speaks **different protocols on each hop**:

- Client connects with HTTP/1.1, Flow uses h2c to backend ✅
- Client uses HTTP/2 (over TLS), Flow uses HTTP/1.1 to backend ✅
- Client uses HTTP/2, Flow uses h2c to backend ✅

Protocol mismatch is fine — Flow translates between them. The only constraint: if a backend is h2c, it must accept HTTP/2 cleartext on a pre-agreed port (no ALPN).

## Performance Characteristics

Throughput depends on three factors:

1. **Encryption** — TLS vs cleartext (encryption costs ~−29% throughput)
2. **Protocol** — HTTP/1.1 vs HTTP/2 (HTTP/2 multiplexing gains ~+51% vs HTTP/1.1)
3. **Multiplexing scope** — client hop only vs both hops (end-to-end multiplexing gains the most)

**Measured throughput (2 CPU cores)**:
- HTTP/1.1, cleartext: ~18,500 req/s
- HTTP/1.1, TLS: ~14,300 req/s
- HTTP/2, TLS (both hops): ~20,000 req/s
- h2c (client h2c, backend HTTP/1.1): ~22,700 req/s
- h2c (both hops): **~28,000 req/s** (fastest)

For detailed latency percentiles, memory, and CPU use at each throughput, see [Performance](performance.md).

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

- **Choosing a protocol?** See [Design Decisions](design-decisions.md) for scenarios and trade-offs.
- **Configuring Flow?** See [Configuration Reference](configuration.md) for complete KDL syntax.
- **Evaluating throughput?** See [Performance](performance.md) for detailed benchmarks and capacity planning.
- **Understanding constraints?** See [Architectural Constraints](architecture-constraints.md) for stream limits and other limits.
