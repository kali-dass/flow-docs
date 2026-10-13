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

**Status**: Both active health checking and circuit breaking are shipped.

- **Active health checking** — see [Configuration → Health checking](configuration.md#health-checking) for syntax and options.
- **Circuit breaking** — see [Configuration → Circuit breaking](configuration.md#circuit-breaking) for syntax and options.

For the difference between them, see [Design Decisions → How Should I Handle Backend Failures?](design-decisions.md#how-should-i-handle-backend-failures).

---

## Next Steps

- **Want to work around these?** See [Design Decisions](design-decisions.md) for architecture patterns that avoid these needs
- **Need a different feature?** This roadmap is not exhaustive — reach out with your use case
- **See also:** [Known Behaviors](known-behaviors.md) for behaviors that aren't missing features but important quirks
