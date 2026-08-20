# Flow Documentation

**Flow** is a high-performance API and AI Gateway built in Rust. Today it functions as an advanced reverse proxy for HTTP/1.1 and HTTP/2 traffic, routing based on paths, headers, and SNI, with load balancing, TLS termination, rate limiting, and header rewriting — all driven by a single KDL config file. Flow is **evolving into a fully-featured API Gateway and AI Gateway** with request transformation, authentication, per-user rate limiting, and LLM traffic management planned for upcoming releases.

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
| [Configuration](configuration.md) | Full KDL reference — system, services, listeners, connectors |
| [Command line](cli.md) | Every flag, and how the CLI and config file interact |
| [Routing](routing.md) | Matching requests to routes, and how ties are broken |
| [Protocols](protocols.md) | HTTP/1.1, HTTP/2, h2c — technical details and performance |
| [Policies](policies.md) | Rate limiting, IP blocking, header rewriting, timing |
| [Performance](performance.md) | Benchmarks, safe operating rates, and tuning |
| [Operations](operations.md) | Running in production, zero-downtime upgrades, logging |

### Understanding Limitations
| Guide | What it covers |
|---|---|
| [Limitations](limitations.md) | What Flow can't do yet, architectural constraints, known behaviors |

### Comparisons
| Guide | What it covers |
|---|---|
| [Comparisons](comparisons/README.md) | Measured against other gateways on identical hardware |

## Current Features

- **Flexible routing** — path prefix, regex, headers, SNI with AND/OR combinators
- **Load balancing** — round-robin, random, consistent hashing (FVN, Ketama)
- **Protocols** — HTTP/1.1, HTTP/2, and cleartext h2c (both client and upstream)
- **TLS** — terminate on clients, originate to upstreams, per-upstream CAs
- **Policies** — rate limiting (global/per-IP/per-path), IP blocking, header rewriting
- **Zero-downtime upgrades** — graceful reload with connection handoff

## Planned Features (API Gateway & AI Gateway)

- **Authentication** — JWT, API-key, OAuth2/OIDC, mTLS, HMAC signing
- **Transformation** — request/response body rewriting, schema validation, CORS, URL rewrites
- **Per-user Rate Limiting** — move beyond per-IP to per-developer/per-tenant control
- **API Versioning** — built-in /v1/ vs /v2/ branching logic
- **LLM Traffic Management** — cost estimation, token counting, usage tracking

These features are planned for upcoming releases. Check back for updates as Flow evolves.
