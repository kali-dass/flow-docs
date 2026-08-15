# Features Not Yet Supported

> **See also:** [Limitations Hub](limitations.md) • [Known Behaviors](known-behaviors.md) • [Architectural Constraints](architecture-constraints.md) • [What Flow Is Not](scope.md)

The features on this page are not yet implemented in Flow but are under consideration. If you need one of these, consider workarounds or reach out to discuss roadmap priorities.

## WebSocket Proxying

**Status**: Not supported

Flow passes through WebSocket upgrade requests (HTTP 101 Upgrade) but does not implement the WebSocket protocol itself. The connection gets passed through as-is, which works if both client and server agree on WebSocket, but Flow provides no special WebSocket support (no message fragmentation handling, no ping/pong, no subprotocol negotiation).

**Workaround**:
- Use a dedicated WebSocket proxy in front of or behind Flow
- Keep WebSocket traffic on a separate port/service that doesn't go through Flow
- Evaluate whether you actually need bidirectional communication (many WebSocket use cases can be replaced with HTTP polling or Server-Sent Events)

**Why not supported**:
- WebSocket is a stateful, bidirectional protocol; Flow is optimized for request/response HTTP
- Significant code complexity for a feature most users don't need

## gRPC

**Status**: Partially supported, not validated

Flow can proxy gRPC traffic because gRPC runs on HTTP/2 (or HTTP/1.1 with custom framing). Flow's HTTP/2 support means gRPC requests should flow through, but we have not validated against real gRPC clients and servers. Edge cases around trailers, custom headers, and flow control may not work correctly.

**Workaround**:
- Test gRPC through Flow in your environment first (don't assume it works)
- Use a dedicated gRPC proxy (Envoy, gRPC LB) for gRPC-only traffic
- Run Flow only for HTTP/REST, keep gRPC separate

**Validation roadmap**:
- We plan to validate against gRPC Go client library and common gRPC services
- Will document known issues and workarounds

## Upstream Health Checking

**Status**: Not supported

Flow has no built-in health checking. If a backend becomes unhealthy or goes down, Flow continues sending it requests until it times out or closes the connection. Requests to unhealthy backends fail (502 Bad Gateway).

**Workaround**:
- **Use external health checking**: Keep only healthy backends in your service discovery system or DNS. Update Flow's configuration when backends change.
- **Rely on client retries**: Clients should retry on 502/503 responses.
- **Layer health checking outside Flow**: Use Kubernetes liveness probes, consul, or another service mesh to manage backend health. Update Flow's connector list dynamically.
- **Accept the tradeoff**: If your backends are stable and failures are rare, the impact may be tolerable.

**Why not supported**:
- Health checking requires persistent state and periodic checks (adds complexity and CPU overhead)
- In containerized environments, orchestrators (Kubernetes) already handle health and scheduling
- For traditional deployments, external load balancers or DNS often provide health checking

**Future roadmap**:
- Passive health checking (fail-open on repeated connection errors) may be considered before full active health checking

---

## Next Steps

- **Want to work around these?** See [Design Decisions](design-decisions.md) for architecture patterns that avoid these needs
- **Need a different feature?** This roadmap is not exhaustive — reach out with your use case
- **See also:** [Known Behaviors](known-behaviors.md) for behaviors that aren't missing features but important quirks
