# Known Behaviors

> **See also:** [Limitations Hub](limitations.md) • [Architectural Constraints](architecture-constraints.md) • [Not Yet Supported](not-yet-supported.md) • [What Flow Is Not](scope.md)

These behaviors are not bugs—they're how Flow is designed to work. Knowing them helps you understand why Flow behaves a certain way and how to design around it.

## All Backends Fail: 502 Bad Gateway

**Behavior**: If all backends in a route become unreachable or close connections, Flow returns HTTP 502 Bad Gateway to the client.

**Why this happens**: Flow has no health checking. When a request arrives:
1. Flow picks a backend (round-robin, random, or hashing)
2. Flow attempts to connect and proxy the request
3. If the connection fails (refused, timeout, closed), Flow returns 502

**No graceful degradation**: There's no "if a few backends fail, route to the healthy ones" logic. All backends must be healthy.

**When you'll see this**:
- All backends are down (maintenance, crash, deployment issue)
- Network partition isolates Flow from backends
- Load balancer upstream of Flow doesn't remove failed backends

**How to handle**:
- **Upstream orchestration**: Use Kubernetes, Consul, or DNS to ensure Flow only sees healthy backends
- **Client retry logic**: Clients should retry on 502 with exponential backoff
- **Monitoring**: Alert when Flow starts returning 502s (indicates backend failure)
- **Graceful shutdown**: During backend maintenance, remove it from Flow's configuration before shutting it down

**Example error**:
```
$ curl -v http://localhost:8080/api/
< HTTP/1.1 502 Bad Gateway
```

---

## HTTP/2 Protocol Mismatch: 502 Bad Gateway

**Behavior**: If a backend is configured with `proto "h2-only"` but negotiates HTTP/1.1 instead, Flow returns 502.

**Why this happens**: Flow enforces strict protocol compliance. If you tell Flow "this backend speaks h2-only" and the backend doesn't deliver, it's a misconfiguration. Silent fallback would mask the issue in production.

**When you'll see this**:
- Backend configured as `h2-only` but doesn't actually support HTTP/2
- Backend's TLS certificate doesn't include HTTP/2 in ALPN
- Backend's cipher suite doesn't support TLS + HTTP/2
- Network misconfiguration causes ALPN negotiation to fail

**How to fix**:
- Verify the backend supports HTTP/2 (test with `curl -I --http2 https://backend:443`)
- Check the backend's certificate and cipher suite
- If HTTP/2 is optional, use `proto "h2-or-h1"` to allow fallback:

```kdl
upstream {
  proto "h2-or-h1"  # Prefer HTTP/2, accept HTTP/1.1
  tls-sni "backend.example.com"
}
```

**No silent fallback**: Unlike some proxies, Flow doesn't silently accept HTTP/1.1 when h2-only is requested. This strictness is intentional.

---

## Rate Limiting: Per-Flow Instance, Not Global

**Behavior**: If you configure rate limiting on a route, the limit applies per Flow instance, not across all instances.

**Example**:
- You set rate limit: 100 req/sec per route
- You run 2 Flow instances
- Client load balances across both instances
- Each instance independently enforces 100 req/sec
- **Total allowed**: ~200 req/sec (100 per instance), not 100 req/sec

**Why this matters**: 
- Rate limiting is in-memory bucket counters
- No shared state between Flow instances
- Prevents inter-process communication overhead

**When you'll notice**:
- Expecting strict rate limiting across your fleet
- Running multiple Flow instances behind a load balancer

**How to handle**:
- **Option 1**: Run a single Flow instance with multiple threads/cores (if throughput allows)
- **Option 2**: Use external rate limiting (Redis) — **or wait for distributed rate limiting** which is planned for Flow
- **Option 3**: Accept that rate limiting is per-instance and account for it in your design

**Roadmap**: Distributed rate limiting is planned for a future release to enable shared rate limit buckets across multiple Flow instances.

---

## Connection Reuse and Pooling

**Behavior**: Flow maintains a connection pool to each upstream backend. Connections are reused across requests when possible.

**Connection reuse**:
- **HTTP/1.1**: Connections use Keep-Alive. If the backend closes the connection, Flow opens a new one on the next request.
- **HTTP/2**: Multiplexing means multiple requests share one connection. Flow maintains the connection for subsequent requests.

**When connections close and reopen**:
- Backend closes connection (idle timeout, explicit close)
- Network interruption
- Backend running out of file descriptors
- Force-killed process

**What you might observe**:
- Occasional 502 errors (connection refused while a new connection opens)
- Latency spike on first request to a backend after connection cycle
- Memory usage growth proportional to concurrent connections

**How to tune** (mostly fixed in Flow's architecture):
- **Idle timeout**: Backends and Flow close idle connections after ~60 seconds
- **Max concurrent connections per backend**: No explicit limit; limited by file descriptor ulimit
- **TCP keep-alive**: Enabled, prevents stale connection buildup

---

## No Request Buffering (Streaming)

**Behavior**: Flow streams request and response bodies without buffering entire messages in memory.

**Why this matters**:
- Large file uploads/downloads don't consume proportional memory
- No request size limit (unlimited file uploads)
- Latency is low (streaming starts immediately)

**Caveats**:
- If the client sends data slowly, Flow waits (streaming is not "fire and forget")
- If backend closes connection mid-stream, client gets a partial response or timeout
- Some policies (e.g., request inspection) require buffering; Flow doesn't do this

**When relevant**: File transfer (large uploads, media serving), long-running requests, streaming APIs

---

## No Request Transformation

**Behavior**: Flow proxies requests as-is (after applying policies). It doesn't transform request content (JSON, query params, etc.).

**What Flow doesn't do**:
- Rewrite request body (e.g., JSON transformation)
- Add/remove query parameters programmatically
- Decompress/recompress bodies
- Cache responses

**What Flow does do**:
- Rewrite headers (add, remove, modify)
- Route based on headers, paths, SNI
- Rate limit, IP block, apply policies

**When you need transformation**: Request and response transformation is **planned for Flow** and coming soon. Today, Flow passes requests through unchanged — transformation will be built in as a native policy. See [API Gateway Roadmap](scope.md#api-gateway-roadmap).

---

## Logging and Observability

**Current behavior**: Flow logs startup, configuration messages, and errors to stdout by default (level and file output are configurable — see [Configuration](configuration.md#loggingapp-log)).

**What's logged today**:
- Startup and configuration messages
- Errors (connection failures, invalid requests, timeouts)
- Warnings (e.g., unverified certs, cleartext backends)

**What's not logged**:
- Per-request logs (structured JSON access logs are **planned**)
- Per-request latencies as log records (available via `timing-header` filter or [planned metrics](operations.md#planned-comprehensive-logging--observability))
- Request/response bodies
- Full debug traces

**Planned for future releases**:
- **Access logs** — structured JSON logs (one per request), with sampling to avoid I/O overhead
- **Metrics** — in-process histograms exposed on `/metrics` for Prometheus
- **Distributed tracing** — OpenTelemetry integration for end-to-end visibility
- **Log masking** — automatic redaction of sensitive headers, tokens, and PII

Until these features ship, use:
- `timing-header` filter for per-request latency in response headers
- `--log-level debug` for startup diagnostics
- External observability tools (APM agents, service mesh) for fleet-wide visibility

**See**: [Operations](operations.md) for current logging configuration and planned features.

---

## Shutdown and Graceful Reload

**Behavior**: Flow supports graceful shutdown and zero-downtime reloads.

- **Graceful shutdown**: When signaled, Flow stops accepting new connections but lets in-flight requests complete (with timeout)
- **Zero-downtime reload**: Flow can pass listening sockets to a new process and shut down old one without dropping connections

**Timeout**: Default graceful shutdown timeout is 30 seconds. In-flight requests that don't complete in time are terminated (client sees connection close or partial response).

**See**: [Operations](operations.md) for upgrade and shutdown procedures.

---

## License Expiration Is Checked at Startup, Not Continuously

**Behavior**: Flow validates its license file once, at startup, before opening any listener. If
the license is invalid or already expired at that point, Flow refuses to start.

**What this means in practice**: if a license expires *while Flow is already running*, Flow
does not notice — it keeps serving traffic normally until the next restart. There is currently
no background check that stops a running Flow instance partway through its process lifetime
because its license expired.

**Planned**: a continuous runtime check that starts rejecting new requests once a license
expires (letting in-flight requests finish normally) is planned but not yet implemented.

**See**: [Licensing](license.md) for the full startup license check and where Flow looks for
the license file.

---

## Next Steps

- **Design around these behaviors?** See [Design Decisions](design-decisions.md) for architecture patterns
- **Operating Flow?** See [Operations](operations.md) for deployment, monitoring, and graceful shutdown
- **Understanding other limitations?** See [Architectural Constraints](architecture-constraints.md) or [Not Yet Supported](not-yet-supported.md)
