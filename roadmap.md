# Roadmap

This is the one place that states what Flow can do today and what is planned. Other pages describe how things behave and link here for status.

Flow is a licensed product: running it requires a valid license file. See [Licensing](license.md).

Flow is an advanced reverse proxy today. The planned direction is an API gateway first, then an LLM gateway, an MCP gateway, an agent-to-agent (A2A) gateway, a model catalog, and registries for MCP servers, A2A agents, and skills. Each builds on the same foundation: the routing, security, and observability below.

**Statuses**

| Status | Meaning |
|---|---|
| Available | Shipped and supported. |
| Preview | Shipped, usable, and may change before it is final. |
| Planned | On the roadmap, not yet shipped. Dates are not committed. |
| Evaluating | Being assessed. It may become Planned, and there is no commitment to build it. |

For what changed in each build, see the [Changelog](changelog.md).

## Gateways

| Capability | Status | Notes |
|---|---|---|
| Advanced reverse proxy | Available | The foundation for everything below: routing, load balancing, security, and observability. |
| API gateway | Planned | The capabilities marked Planned in the sections below. |
| LLM gateway | Planned | Token-aware rate limiting and spend limits, guardrails, model routing, provider fallover, AI response caching. |
| Model catalog | Planned | The list of models that are deployed and configured, which routing, fallover, spend limits and rate limits refer to. |
| MCP gateway | Planned | Session-aware routing, tool-level access control, per-session rate limits, MCP-aware guardrails. |
| MCP server registry | Planned | Where teams register their MCP servers for discovery and governance. |
| Agent-to-agent (A2A) gateway | Planned | Iteration budgets and task correlation across agent delegations. |
| A2A agent registry | Planned | Where teams register their agents for discovery and governance. |
| Skills registry | Planned | |

## Routing and load balancing

| Capability | Status | Notes |
|---|---|---|
| Routing by path, header, or TLS SNI | Available | [Routing](routing.md) |
| Load balancing (round-robin, random, consistent hashing) | Available | [Configuration](configuration.md) |
| Hostname upstreams with background DNS refresh | Available | [DNS Resolution](dns.md) |
| DNS refresh that follows the record TTL | Planned | The interval is fixed today. |
| Active health checking | Available | [Health checking](policies/health-checking.md) |
| Circuit breaking | Available | [Circuit breaking](policies/circuit-breaker.md) |
| Weighted and canary traffic splitting | Planned | |
| Traffic mirroring to a shadow upstream | Planned | |

## Protocols

| Capability | Status | Notes |
|---|---|---|
| HTTP/1.1 and HTTP/2, client side and upstream | Available | [Protocols](protocols.md) |
| Cleartext HTTP/2 (h2c) to upstreams | Available | [Protocols](protocols.md) |
| gRPC over cleartext HTTP/2 (unary and streaming) | Available | [Non-REST Protocols](non-rest-protocols.md) |
| gRPC over TLS from clients (TLS terminated at Flow) | Available | [Non-REST Protocols](non-rest-protocols.md#grpcs-tls) |
| gRPC over TLS to the backend | Available | [Non-REST Protocols](non-rest-protocols.md#tls-from-flow-to-the-backend) |
| gRPC-native error responses for errors Flow produces | Planned | Today these reach gRPC clients as HTTP errors. |
| Server-Sent Events (SSE) | Available | [Non-REST Protocols](non-rest-protocols.md) |
| WebSocket passthrough | Preview | The upgrade passes through; not yet validated against real WebSocket clients. See [Non-REST Protocols](non-rest-protocols.md). |
| HTTP/3 (QUIC) | Planned | |
| gRPC transcoding (REST and JSON clients calling gRPC backends) | Evaluating | |

## Security and authentication

| Capability | Status | Notes |
|---|---|---|
| TLS termination and upstream TLS | Available | [Configuration](configuration.md) |
| Token authentication (JWT and opaque tokens) | Available | [Token authentication](policies/token-auth.md) |
| IP and CIDR restriction | Available | [IP restriction](policies/ip-restriction.md) |
| API-key authentication | Planned | |
| OAuth2 / OIDC authorization code flow | Planned | |
| External authorization (forward-auth, OPA) | Planned | |
| mTLS with client certificates, both directions | Planned | |
| Forwarding verified identity as upstream headers | Planned | |
| Request validation against an OpenAPI schema | Planned | |
| Injection and bot protection, JSON and XML threat limits | Planned | |

## Traffic control

| Capability | Status | Notes |
|---|---|---|
| Rate limiting per instance (global, per IP, per URI, per client, claim-based tiers) | Available | [Rate limiting](policies/rate-limiting.md) |
| Rate limiting shared across Flow instances | Planned | Limits are per instance today. |
| Quotas (daily and monthly caps) | Planned | |
| Request termination (fixed status, maintenance responses) | Available | [Request termination](policies/request-termination.md) |
| Redirects (HTTPS enforcement, path migration) | Planned | |
| CORS handling | Planned | |
| Request and response size limits | Planned | |

## Transformation and caching

| Capability | Status | Notes |
|---|---|---|
| Request and response header rewriting | Available | [Upsert header](policies/upsert-header.md), [Remove header](policies/remove-header-key-regex.md) |
| Correlation ID | Available | [Correlation ID](policies/correlation-id.md) |
| Response caching (in memory, per instance) | Available | [Response caching](policies/cache-response.md) |
| URI and path rewriting | Planned | |
| Request and response body transformation | Planned | Bodies pass through unchanged today. |

## Observability

| Capability | Status | Notes |
|---|---|---|
| Timing headers | Available | [Timing header](policies/timing-header.md) |
| Prometheus metrics endpoint | Preview | [Metrics](metrics.md) |
| Per-backend and per-policy metrics | Planned | |
| TLS and authentication for the metrics endpoint | Planned | |
| Structured access logs with external sinks | Planned | |
| OpenTelemetry tracing | Planned | |
| Log masking for secrets and personal data | Planned | |

## Operations

| Capability | Status | Notes |
|---|---|---|
| Single configuration file with validation before start | Available | [CLI](cli.md) |
| Zero-downtime upgrade | Available | [Operations](operations.md) |
| Reload configuration without replacing the process | Planned | A configuration change needs a restart or an upgrade today. See [Configuration changes](behavior.md#configuration-changes). |
| TLS certificate rotation without replacing the process | Planned | Listener certificates are read at startup. See [Configuration changes](behavior.md#configuration-changes). |
| Configurable shutdown timeout that exits once connections drain | Planned | The grace period is fixed at 5 minutes today. See [Operations](operations.md#stopping-flow). |
| Continuous license check while running | Planned | The license is verified at startup today. See [Licensing](license.md). |
