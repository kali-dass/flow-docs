# Where Flow Fits

> **See also:** [How Flow Behaves](behavior.md) • [Roadmap](roadmap.md) • [FAQ](faq.md)

Flow is an advanced reverse proxy growing into a gateway for APIs, LLMs, MCP servers, and AI agents. This page shows where it sits in an architecture and what to run alongside it. For what is available and what is planned, see the [Roadmap](roadmap.md).

---

## Service mesh

**Service meshes** (Istio, Linkerd, Consul Connect) manage traffic between microservices through a sidecar proxy next to every workload.

Flow is a **central gateway**, not a sidecar. Backends are configured explicitly, with DNS names re-resolved in the background, and TLS is between Flow and each backend, not enforced mesh-wide.

**Choose Flow when**:
- You have a central gateway and a set of backend services
- You want simplicity over full mesh observability
- You need HTTP/2 multiplexing and very high throughput at the edge
- You need token authentication and traffic policies at the gateway layer

**Choose a service mesh when**:
- You run hundreds of microservices with complex inter-dependencies
- You need automatic service discovery and load balancing between every service pair
- You need distributed tracing and metrics for every internal hop
- You need mTLS enforced automatically between all services

**Together:** run Flow as the ingress gateway in front of the mesh. Flow handles external traffic and API policies; the mesh handles internal traffic.

---

## Load balancer

**Traditional load balancers** (F5, AWS ELB, Citrix) distribute traffic across server pools, with geographic routing and cross-region failover.

Flow load balances HTTP traffic across the backends of a route:
- HTTP/2-aware balancing: round-robin, random, and consistent hashing
- DNS name resolution for upstream hostnames, with background refresh
- Active health checking: `health-check "Active"` in the `load-balance` block
- Circuit breaking: `circuit-breaker { }` inside any `load-balance` block

**Choose Flow when**:
- You need HTTP routing by path, header, or SNI, not only L4 distribution
- You need token authentication and rate limiting next to routing
- Your orchestrator (for example Kubernetes) handles backend health and auto-scaling

**Put a load balancer in front of Flow for**:
- Multi-region failover across data centers
- Non-HTTP protocols such as raw TCP and UDP
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

## WAF

**Web Application Firewalls** (Cloudflare WAF, ModSecurity, AWS WAF) inspect requests for threats such as SQL injection, XSS, the OWASP Top 10, bot traffic, and volumetric DDoS.

Flow protects the traffic it routes with:
- IP and CIDR restriction
- Rate limiting per IP, per path, or per client
- Bearer token authentication before routing

For full WAF protection, place a managed WAF in front of Flow:

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

> **Planned:** request validation, injection and bot protection, and JSON and XML threat limits. See [Roadmap](roadmap.md#security-and-authentication).

---

## Caches and CDNs

Flow caches responses in memory on each instance; see [Cache response](policies/cache-response.md). Run a shared cache or a CDN alongside Flow when you need a cache shared across instances or kept across restarts. The same page shows how.

---

## What to use with Flow

Flow sits between the network edge and your backends.

```
Internet
  |
  v
[CDN or WAF] (optional: caching, DDoS, bot protection)
  |
  v
[Load Balancer] (optional: geographic failover)
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

---

## Next steps

- **Is Flow right for your use case?** See [Design Decisions](design-decisions.md)
- **What does Flow do in detail?** See [Technical Overview](technical-overview.md)
- **Which policies are available?** See [Policies](policies.md)
- **Performance and capacity?** See [Performance](performance.md)
- **What is released?** See [Changelog](changelog.md)
