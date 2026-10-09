# Flow Documentation

**Flow** is a high-performance API and AI Gateway built in Rust. Today it handles routing, TLS termination, HTTP/2 multiplexing, token authentication (JWT and opaque tokens), rate limiting, IP blocking, and header policies — all from a single KDL config file. Flow is **evolving into a fully-featured API and AI Gateway** across four planned phases: API Gateway, LLM Gateway, MCP Gateway, and A2A Gateway.

> **Pre-release software.** Flow has not yet had a stable release. Configuration syntax, defaults, and behavior — including for features already documented here — may change without notice between builds as the product evolves. Once Flow reaches a stable release, changes that break existing configurations will be called out explicitly and follow normal versioning practice.

## Quick Start

Choose where to start based on what you need:

- **New to Flow?** Start with [Getting Started](getting-started.md)
- **Want to understand how it works?** See [Technical Overview](technical-overview.md)
- **Making architecture decisions?** See [Design Decisions](design-decisions.md)
- **Understanding what Flow can/can't do?** See [Limitations](limitations.md)
- **Comparing with other gateways?** See [Comparisons](comparisons/README.md)
- **Running in production?** See [Operations](operations.md)

## Documentation

### Getting Started
| Guide | What it covers |
|---|---|
| [Getting started](getting-started.md) | Install, run, and verify your first proxy |
| [Technical Overview](technical-overview.md) | How Flow works, core concepts, request lifecycle |
| [Design Decisions](design-decisions.md) | Choose protocol, load balancing, policies, routing patterns |

### Reference
| Guide | What it covers |
|---|---|
| [Configuration](configuration.md) | Full KDL reference — system, logging, services, listeners, connectors |
| [Command line](cli.md) | Every flag, and how the CLI and config file interact |
| [Routing](routing.md) | Matching requests to routes, and how ties are broken |
| [Protocols](protocols.md) | HTTP/1.1, HTTP/2, h2c — technical details and performance |
| [Policies](policies.md) | Rate limiting, IP blocking, header rewriting, timing |
| [Performance](performance.md) | Benchmarks, safe operating rates, and tuning |
| [Operations](operations.md) | Running in production, zero-downtime upgrades, logging |
| [Licensing](license.md) | The license file, where Flow looks for it, and what happens at expiration |

### Understanding Limitations
| Guide | What it covers |
|---|---|
| [Limitations](limitations.md) | What Flow can't do yet, architectural constraints, known behaviors |

### Comparisons
| Guide | What it covers |
|---|---|
| [Comparisons](comparisons/README.md) | Measured against other gateways on identical hardware |

### Changelog
| Guide | What it covers |
|---|---|
| [Changelog](changelog.md) | Release notes — what shipped and when, by version |

## Current Features

- **Flexible routing** — path prefix, regex, headers, SNI with AND/OR combinators
- **Load balancing** — round-robin, random, consistent hashing (FVN, Ketama)
- **Protocols** — HTTP/1.1, HTTP/2, and cleartext h2c (both client and upstream)
- **TLS** — terminate on clients, originate to upstreams, per-upstream CAs
- **Token authentication** — [JWT (JWKS or static key, multi-issuer) or opaque token via RFC 7662 introspection](policies/token-auth.md)
- **Traffic policies** — rate limiting (global/per-IP/per-path), IP blocking, header rewriting, timing headers
- **DNS name resolution** — upstream connectors accept hostnames with background refresh
- **Zero-downtime upgrades** — graceful reload with connection handoff

## Planned Features

See [What Flow Is](scope.md) for the full four-phase roadmap. Highlights by phase:

**Phase 1 — API Gateway** (next release):
- API-key, OAuth2/OIDC, mTLS, HMAC, forward-auth, OPA authorization
- URI/path rewriting, CORS, redirect, request-ID injection, shadow traffic mirroring
- OpenAPI / JSON Schema request validation
- Response caching (TTL-based, in-process)
- Weighted/canary traffic splitting and circuit breaking
- Per-user/per-tenant rate limiting and quota enforcement
- Prometheus metrics endpoint, OpenTelemetry tracing, structured access logs

**Phase 2 — LLM Gateway**: token rate limiting, guardrails, model routing, spend limits, AI response caching, provider fallover

**Phase 3 — MCP Gateway**: session-aware routing, tool-level access control, MCP guardrails

**Phase 4 — A2A Gateway**: iteration budget enforcement, agent task correlation

See [Changelog](changelog.md) for what has shipped.
