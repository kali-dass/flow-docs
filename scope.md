# What Flow Is Not

> **See also:** [Limitations Hub](limitations.md) • [Architectural Constraints](architecture-constraints.md) • [Known Behaviors](known-behaviors.md) • [Not Yet Supported](not-yet-supported.md)

Understanding what Flow is **not** helps you evaluate whether it's the right tool and design architectures that complement it correctly.

## Flow Today: A High-Performance Reverse Proxy

Flow is currently a **high-performance reverse proxy**. It accepts HTTP/1.1 and HTTP/2 requests from clients, routes them to upstream backends, and returns responses. However, Flow is **evolving to become a full API Gateway and AI Gateway** — these are planned capabilities, not distant ideas.

### What That Means

✅ **Flow does**:
- Route HTTP requests based on path, headers, or SNI
- Multiplex over HTTP/2 (client and upstream)
- Terminate/originate TLS connections
- Rewrite request/response headers
- Rate limit and IP block traffic
- Run multiple services on different ports
- Zero-downtime reloads

❌ **Flow doesn't**:
- Parse or transform request/response bodies
- Cache responses
- Compress/decompress
- Buffer requests before proxying
- Inspect for security threats (WAF)
- Authenticate users
- Do service discovery (auto-find backends)
- Check backend health

## Not a Service Mesh

**Service meshes** (Istio, Linkerd, Consul Connect) are infrastructure tools that manage all traffic between microservices. They typically work via sidecars (one proxy per pod/container).

**Flow is not a service mesh because**:
- No sidecar per pod — Flow is a central gateway
- No automatic service discovery — backends must be configured statically
- No distributed tracing built-in — you add it yourself
- No mutual TLS enforcement across the mesh — only Flow↔backend TLS
- No circuit breaker or retry logic built-in

**When to use Flow instead of a service mesh**:
- You have a central gateway and a few backend services
- You want simplicity over full mesh observability
- You're not running microservices-heavy infrastructure
- You need HTTP/2 multiplexing and ultra-high throughput

**When to use a service mesh**:
- Hundreds of microservices with complex inter-dependencies
- You need automatic service discovery and load balancing
- You need distributed tracing and metrics per service-pair
- You need mTLS between all services automatically

**Can Flow work with a service mesh?** Yes. You can run Flow as an ingress gateway in front of a service mesh. The mesh handles internal traffic; Flow handles external traffic.

---

## Not a Load Balancer

**Load balancers** (F5, Citrix, AWS ELB) distribute traffic across multiple servers and provide health checking, failover, and cross-region routing.

**Flow is not a load balancer because**:
- No active health checking — can't detect and avoid failed backends
- No failover — if all backends fail, returns 502
- No geographic routing — can't route based on location
- No DNS-based routing — can't respond to DNS queries

**When to use Flow instead**:
- You're deploying to a single region/zone
- Orchestrator (Kubernetes) handles health/failover outside Flow
- You need HTTP request routing (not just L4 load balancing)

**When to use a load balancer**:
- Multi-region failover across data centers
- Active health checking with automatic backend removal
- Non-HTTP protocols (TCP, UDP)
- Geographic routing (route by IP geolocation)

**Can Flow work with a load balancer?** Yes. Use a traditional load balancer for infrastructure-level failover and geographic routing, then route to Flow for HTTP request routing.

```
Internet
    |
    v
[Traditional Load Balancer] (health check, DNS failover)
    |
    v
[Flow 1] [Flow 2] [Flow 3] (HTTP routing)
    |
    v
[Backends]
```

---

## Not a WAF

**Web Application Firewalls** (Cloudflare, ModSecurity, AWS WAF) inspect requests for security threats (SQL injection, XSS, bot detection, rate limiting, DDoS mitigation).

**Flow is not a WAF because**:
- No request inspection (doesn't parse request bodies or cookies for threats)
- No OWASP rule sets
- No bot detection
- No DDoS mitigation (basic rate limiting only)
- No geo-blocking built-in

**Flow can do**:
- IP/CIDR blocking (basic network-level blocking)
- Header-based routing (route suspicious patterns)
- Basic rate limiting (per-IP or per-path)

**When you need WAF protection**:
- Public-facing APIs exposed to untrusted internet traffic
- Web applications vulnerable to injection attacks
- APIs that must reject bots/scrapers
- DDoS protection requirements

**How to combine Flow + WAF**:

```
Internet
    |
    v
[WAF] (threat detection, rate limiting)
    |
    v
[Flow] (HTTP routing)
    |
    v
[Backends]
```

Or use a managed WAF service (Cloudflare, AWS WAF) in front of Flow.

---

## Not a Cache

**Caches** (Varnish, CDN, Redis) store responses and serve them on subsequent identical requests.

**Flow doesn't cache** because:
- No response caching
- No cache invalidation logic
- No ETags or cache headers handling
- Passes every request to the backend

**When you need caching**:
- Reduce backend load (frequently accessed, slow-to-compute responses)
- Improve latency for repeated requests
- Static content serving (images, CSS, JS)

**How to combine Flow + Cache**:

```
Flow
  |
  v
[Cache Layer] (Varnish, CDN, Redis)
  |
  v
[Backends]
```

Or use a CDN for static content:

```
Internet
  |
  +---> [CDN] (cached static content)
  |
  +---> [Flow] (dynamic requests)
            |
            v
        [Backends]
```

---

## API Gateway Roadmap

**Today**: Flow is a high-performance reverse proxy focused on routing and load balancing. It handles HTTP/1.1 and HTTP/2 traffic, TLS termination, policies (rate limiting, header rewriting), and [JWT validation](policies/jwt-validate.md) (JWKS or a static key, multi-issuer).

**Planned**: Flow is evolving to include **full API Gateway** and **AI Gateway** capabilities. These will bring:
- API versioning (built-in /v1/ vs /v2/ branching logic)
- Per-developer/per-user rate limits and quotas
- Request/response transformation (schema validation, body rewriting)
- Further authentication methods (API-key, OAuth2/OIDC, mTLS enforcement, HMAC signing)
- API documentation and schema management
- AI/LLM traffic handling and cost control

**Current gaps** (today's Flow):
- No built-in API versioning
- Rate limiting is per-IP or per-path only (not per-user/developer)
- No request/response body transformation
- JWT validation is supported; OAuth2/OIDC, API-key, and mTLS auth are not yet
- No API documentation/schema management

**Coming soon to Flow**:
- API versioning and per-user/per-developer rate limiting
- Request/response transformation and validation
- OAuth2/OIDC, API-key, and mTLS authentication (JWT validation already shipped)
- API documentation and schema management

Flow is evolving to consolidate API management into a single, high-performance system. Check the roadmap as these features ship — the goal is to eliminate the need for layered API gateway solutions.

---

## What to Use With Flow

**Typical architecture**:

```
Internet
  |
  v
[CDN or WAF] (optional, for security/caching)
  |
  v
[Load Balancer] (optional, for geographic failover)
  |
  v
[Flow Cluster] (HTTP routing, TLS termination, load balancing)
  |
  v
[Orchestrator: Kubernetes/Consul/DNS] (backend discovery, health checks)
  |
  v
[Backend Services]
```

**Each layer's responsibility**:
- **CDN/WAF**: Security, caching, DDoS protection
- **Load Balancer**: Geographic routing, cross-region failover (if needed)
- **Flow**: HTTP request routing, TLS termination, multiplexing
- **Orchestrator**: Backend health, discovery, auto-scaling
- **Backends**: Your application logic

---

## Next Steps

- **Is Flow right for your use case?** See [Design Decisions](design-decisions.md)
- **Want to understand what Flow does**? See [Technical Overview](technical-overview.md)
- **Building an architecture around Flow?** See [Design Decisions](design-decisions.md) for patterns
