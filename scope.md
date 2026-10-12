# What Flow Is (and Is Not)

> **See also:** [Limitations Hub](limitations.md) • [Architectural Constraints](architecture-constraints.md) • [Known Behaviors](known-behaviors.md) • [Not Yet Supported](not-yet-supported.md)

Understanding what Flow does and doesn't do today helps you evaluate whether it is the right
tool and design architectures that complement it correctly. Flow is actively evolving — several
items in the "not yet" list are in active development.

---

## Flow Today

Flow is a **high-performance API and AI Gateway** built on a reverse-proxy foundation. Today it
handles routing, TLS, HTTP/2 multiplexing, token authentication, traffic control policies, and
header transformation — all from a single configuration file.

### What Flow does today

- Route HTTP requests based on path, headers, or TLS SNI
- Multiplex over HTTP/2 — both client-side and upstream (including cleartext h2c)
- Terminate and originate TLS connections
- Verify bearer tokens — JWT (JWKS or static key, multi-issuer) or opaque tokens via RFC 7662 introspection
- Strip verified tokens before forwarding to backends
- Rate limit traffic — globally, per client IP, or per URI path
- Block requests from IP/CIDR ranges
- Rewrite request and response headers (add, replace, remove)
- Inject latency timing headers for per-request visibility
- Run multiple services on different ports from one process
- Hot-reload configuration with zero downtime

### What Flow does not do yet

| Capability | Status |
|---|---|
| Parse or transform request/response bodies | Planned — API Gateway Phase 1 |
| Cache responses | Planned — API Gateway Phase 1 |
| Validate requests against an OpenAPI schema | Planned — API Gateway Phase 1 |
| Redirect HTTP → HTTPS or deprecated paths | Planned — API Gateway Phase 1 |
| Add CORS headers | Planned — API Gateway Phase 1 |
| Rewrite request URIs or strip path prefixes | Planned — API Gateway Phase 1 |
| Mirror traffic to a shadow upstream | Planned — API Gateway Phase 1 |
| API-key authentication | Planned — API Gateway Phase 1 |
| OAuth2/OIDC authorization code flow | Planned — API Gateway Phase 1 |
| External auth delegation (forward-auth, OPA) | Planned — API Gateway Phase 1 |
| Per-user/per-tenant rate limiting | Planned — API Gateway Phase 1 |
| Circuit breaking on upstream errors | Planned — API Gateway Phase 1 |
| Weighted / canary traffic splitting | Planned — API Gateway Phase 1 |
| Active upstream health checking | Shipped |
| Prometheus metrics endpoint | Planned — infra |
| Distributed tracing (OpenTelemetry) | Planned — infra |
| Per-request access logging with external sinks | Planned — infra |
| mTLS client certificates (upstream and downstream) | Planned — API Gateway Phase 1 |
| LLM-specific policies (token rate limiting, guardrails, model routing) | Planned — LLM Gateway Phase 2 |
| MCP gateway policies | Planned — MCP Gateway Phase 3 |
| Compress/decompress responses | Not yet scoped |
| Inspect for WAF-style security threats | Not yet scoped |
| Service discovery (auto-find backends) | Not yet scoped |
| Geographic routing | Not yet scoped |
| HTTP/3 (QUIC) | Blocked on upstream dependency |

---

## Not a Service Mesh

**Service meshes** (Istio, Linkerd, Consul Connect) manage all traffic between microservices
via sidecars — one proxy per pod/container.

**Flow is not a service mesh because**:
- No sidecar per pod — Flow is a central gateway
- No automatic service discovery — backends are configured statically (DNS resolution for
  hostnames is supported; dynamic registry sync is not)
- No distributed tracing built-in yet — OpenTelemetry support is planned
- No mesh-wide mTLS enforcement — only Flow↔backend TLS
- No circuit breaker yet — planned in Phase 1

**When to use Flow instead of a service mesh**:
- You have a central gateway and a set of backend services
- You want simplicity over full mesh observability
- You need HTTP/2 multiplexing and ultra-high throughput at the edge
- You need token authentication and traffic policies at the gateway layer

**When to use a service mesh**:
- Hundreds of microservices with complex inter-dependencies
- You need automatic service discovery and load balancing between every service pair
- You need distributed tracing and metrics for every internal hop
- You need mTLS automatically enforced between all services

**Can Flow work with a service mesh?** Yes. Run Flow as the ingress gateway in front of a
service mesh — Flow handles external traffic and API policies; the mesh handles internal traffic.

---

## Not a Traditional Load Balancer

**Traditional load balancers** (F5, AWS ELB, Citrix) distribute traffic across server pools
with active health checking, geographic routing, and cross-region failover.

**What Flow does and doesn't do**:
- ✅ HTTP/2-aware load balancing (round-robin, random, consistent hashing) across configured backends
- ✅ DNS name resolution for upstream hostnames with background refresh
- ✅ Active health checking — `health-check "Active"` in the `load-balance` block
- ⏳ Circuit breaking on upstream errors — **planned in Phase 1**
- ❌ Geographic routing — not planned
- ❌ Cross-region failover — not planned
- ❌ Non-HTTP protocols (TCP, UDP) — not planned

**When to use Flow instead of a load balancer**:
- You need HTTP request routing (path, header, SNI), not just L4 distribution
- You need token authentication and rate limiting co-located with routing
- Your orchestrator (Kubernetes) handles backend health and auto-scaling

**When to use a load balancer in front of Flow**:
- Multi-region failover across data centers
- Non-HTTP protocols
- Geographic routing by IP geolocation

```
Internet
    |
    v
[Load Balancer] (geographic failover, multi-region)
    |
    v
[Flow Cluster] (HTTP routing, auth, rate limiting, policies)
    |
    v
[Backends]
```

---

## Not a WAF

**Web Application Firewalls** (Cloudflare WAF, ModSecurity, AWS WAF) inspect requests for
security threats: SQL injection, XSS, OWASP Top 10, bot detection, DDoS mitigation.

**Flow is not a WAF because**:
- No OWASP rule sets or full threat intelligence
- No DDoS mitigation beyond static rate limiting
- No geo-blocking

**Flow can do today**:
- IP/CIDR blocking
- Rate limiting per-IP or per-path
- Bearer token authentication before routing

**Flow will add** (Phase 1):
- OpenAPI / JSON Schema request validation — rejects structurally malformed requests
- Injection protection — regex-based SQL injection / XSS / command injection detection
- Bot detection — block known scanner and scraper User-Agents
- JSON and XML threat protection — structural size and depth limits
- CORS header enforcement
- Per-user/per-tenant rate limiting (identity-aware, after token auth)

**For full WAF protection**, place a managed WAF (Cloudflare, AWS WAF) in front of Flow:

```
Internet
    |
    v
[WAF] (OWASP, bot detection, DDoS mitigation)
    |
    v
[Flow] (routing, auth, rate limiting, policies)
    |
    v
[Backends]
```

---

## Not a Cache (Yet)

**Caches** (Varnish, CDN, Redis) store responses and serve them on subsequent identical
requests without touching the backend.

**Flow does not cache responses today.** Every request is forwarded to the backend.

**Response caching is planned for API Gateway Phase 1:**
- TTL-based cache for `GET` responses, keyed on method + path + query + optional headers
- `Cache-Control: max-age` / `s-maxage` respected
- Configurable maximum body size and total cache capacity
- Streaming responses (SSE, chunked) are bypassed automatically

**When you need caching today**, add a dedicated cache layer:

```
Flow
  |
  v
[Cache Layer] (Varnish, Redis, CDN)
  |
  v
[Backends]
```

Or use a CDN for static content alongside Flow for dynamic APIs:

```
Internet
  |
  +---> [CDN] (static assets, cached responses)
  |
  +---> [Flow] (dynamic API requests, auth, routing)
```

---

## Roadmap: Four Gateway Phases

Flow is evolving through four planned phases. Each phase adds a complete capability layer on
top of the previous one.

### Phase 1 — API Gateway

The next release. Builds on today's reverse proxy foundation with the policies and
transformations needed to operate a production API gateway:

**Authentication & authorization**
- API-key authentication (static KDL keys and database-backed runtime keys)
- OAuth2/OIDC authorization code flow for browser users
- Forward-auth (delegate auth to an external HTTP service)
- Open Policy Agent (OPA) integration for policy-as-code authorization
- mTLS — client certificates to upstreams and from downstream clients
- Claim injection — forward verified identity fields as upstream headers

**Traffic management**
- Weighted/canary traffic splitting across upstream groups
- Circuit breaking — trip a failing upstream out of rotation automatically
- Redirect policy — HTTP `3xx` from config (HTTPS enforcement, path migration)
- Request-ID injection — unique correlation ID on every request
- CORS — automatic `Access-Control-*` headers and preflight handling
- Per-user/per-tenant rate limiting (identity-aware, builds on token-auth)
- Distributed rate limiting (Redis-backed for multi-instance deployments)
- Quota enforcement — daily/monthly caps per API key or tenant

**Transformation**
- URI and path rewriting (prefix strip, regex rewrite)
- Request/response body transformation (field mapping, format conversion)
- OpenAPI / JSON Schema request validation
- Shadow traffic mirroring to a secondary upstream

**Caching**
- API response cache (TTL-based, in-process, `Cache-Control` aware)

**Observability**
- Prometheus metrics endpoint (`/metrics`) — request counts, latency histograms, error rates
- Per-request access logging with external sinks (HTTP push, Kafka, OpenTelemetry OTLP)
- OpenTelemetry distributed tracing (W3C `traceparent` propagation + OTLP span export)

### Phase 2 — LLM Gateway

AI-specific policies layered on top of the API Gateway:
- `ai-proxy` — normalize requests to OpenAI-compatible LLM providers
- Token rate limiting — bucket by input/output/total token count
- Spend limits — per-key/per-team dollar-cost budget enforcement
- Prompt and response guardrails — keyword/regex blocking, external classifier integration
- Model routing — rewrite `model` field or upstream based on request properties
- AI response caching — exact-match cache for deterministic calls (`temperature=0`)
- Provider fallover — automatic failover on context-window-exceeded or content-policy errors

### Phase 3 — MCP Gateway

Gateway policies for [Model Context Protocol](https://modelcontextprotocol.io/) traffic:
- MCP session-aware routing (sticky by `Mcp-Session-Id`)
- Per-session and per-server rate limiting
- Tool-level access control (restrict which identities can call which tools)
- Tool namespace rewriting (prefix injection to prevent name collisions across MCP servers)
- MCP-aware guardrails (inspect tool call arguments and results)

### Phase 4 — A2A Gateway

Gateway policies for agent-to-agent traffic:
- Iteration budget enforcement — cap multi-turn agent loops per original request
- Task correlation and state tracking across agent delegations

---

## What to Use With Flow

As Flow grows, its role in the architecture expands. Today it sits between the network edge
and your backends; over time it absorbs more of the API management layer.

**Typical architecture today**:

```
Internet
  |
  v
[CDN or WAF] (optional — caching, DDoS, bot protection)
  |
  v
[Load Balancer] (optional — geographic failover)
  |
  v
[Flow] (routing, TLS, auth, rate limiting, header policies)
  |
  v
[Orchestrator: Kubernetes / DNS] (backend health, discovery)
  |
  v
[Backend Services]
```

**Typical architecture with Phase 1 API Gateway**:

```
Internet
  |
  v
[WAF] (optional — OWASP, bot detection)
  |
  v
[Flow] (routing, TLS, auth, rate limiting, caching, transformation,
        circuit breaking, CORS, schema validation, metrics, tracing)
  |
  v
[Backend Services]
```

As Phase 1 ships, several layers that today require separate tools (auth proxies, API
management planes, cache sidecars) collapse into Flow's policy layer.

---

## Next Steps

- **Is Flow right for your use case?** See [Design Decisions](design-decisions.md)
- **What does Flow do in detail?** See [Technical Overview](technical-overview.md)
- **Available policies today?** See [Policies](policies.md)
- **Performance and capacity?** See [Performance](performance.md)
- **What's released?** See [Changelog](changelog.md)
