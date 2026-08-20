# Limitations

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

Flow is a focused reverse proxy. It excels at routing and proxying HTTP/1.1 and HTTP/2 traffic with high throughput. This page explains what Flow can't do and what you should know when building around it.

## Understanding Flow's Limitations

Flow has different types of limitations. Choose what's relevant to your use case:

### [**Features Not Yet Supported**](not-yet-supported.md)

For "Why can't I do X?" — features on the roadmap but not yet implemented:
- WebSocket proxying
- gRPC (implemented, not validated)
- Upstream health checking

### [**Architectural Constraints**](architecture-constraints.md)

For "What are my limits?" — inherent limits from Flow's design when you're in the planning or design phase:
- HTTP/2 stream limits
- Header size limits
- CPU saturation rates
- Memory footprint
- Regex matching limitations

### [**Known Behaviors**](known-behaviors.md)

For "Why is it doing that?" — behaviors that aren't bugs but important to understand:
- How Flow responds when all backends fail
- Protocol enforcement (h2-only strict matching)
- Rate limiting scope (per-instance)
- Connection pooling and reuse
- License expiration checked at startup only, not continuously while running

### [**What Flow Is Not (Yet)**](scope.md)

For "Is Flow right for us?" — clarification on scope for evaluating whether Flow fits your architecture today:
- Not a service mesh
- Not a load balancer (no health checking; **planned for future**)
- Not a WAF
- Not a cache
- Not a full API Gateway today (no versioning, no per-user rate limiting, no transformation; **planned for future**)

---

## Quick Answers

**Can Flow cache responses?** No. Flow is a pass-through proxy, not a cache. Use a caching layer (CDN, Varnish, etc.) upstream if needed.

**Can Flow do gRPC?** It can proxy gRPC traffic (gRPC uses HTTP/2), but we haven't validated it against real gRPC traffic yet. See [gRPC](not-yet-supported.md#grpc).

**Can Flow check if a backend is healthy?** Not yet. All backends must be healthy. See [Upstream Health Checking](not-yet-supported.md#upstream-health-checking).

**Can I rate-limit across multiple Flow instances?** No. Rate limiting is per-instance, in-memory. You need to run a single Flow instance or use external rate limiting. See [Known Behaviors](known-behaviors.md).

**What's the maximum throughput?** Depends on CPU cores and protocol. ~14k–28k req/s on 2 cores depending on encryption and multiplexing. See [Performance](performance.md).

**Can Flow do virtual hosting?** Yes, via SNI matching or header matching. See [Design Decisions](design-decisions.md).

**Can Flow run in Kubernetes?** Yes. Deploy as a DaemonSet or Deployment, expose via a Kubernetes Service. No special integration yet (no automatic service discovery).

**Does Flow need a license to run?** Yes — every instance requires a valid license file, checked at startup before Flow opens any listener. See [Licensing](license.md).

**What observability features does Flow have?** Currently: basic startup logging and response-header timing injection. **Planned for future releases**: structured access logs (JSON, per-request), Prometheus metrics endpoint, OpenTelemetry tracing, and log masking for PII/secrets. See [Operations](operations.md#planned-comprehensive-logging--observability).

---

## Next Steps

- **Want details?** Pick one of the four limitation categories above (Not Yet Supported, Constraints, Behaviors, Scope)
- **Planning a deployment?** See [Design Decisions](design-decisions.md) for architecture guidance that accounts for these constraints
- **Need to work around a constraint?** See [Operations](operations.md) for deployment patterns and [Performance](performance.md) for capacity planning
