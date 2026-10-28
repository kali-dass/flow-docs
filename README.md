# Flow

**A hyper-fast, secure, and extremely lightweight reverse proxy for your services, growing into the gateway and registry for APIs, LLMs, MCP servers, and AI agents.**

Flow sits in front of your applications and handles the traffic work they should not have to: encrypting connections, sending each request to the right place, protecting services from overload, and showing you what is happening. You describe what you want in one configuration file, and Flow does the rest.

Flow is an advanced reverse proxy today. The direction is a full API gateway, then an LLM gateway with a model catalog, an MCP gateway with an MCP server registry, an agent-to-agent (A2A) gateway with an agent registry, and a skills registry. See the [Roadmap](roadmap.md).

> **Pre-release software.** Flow has not had a stable release yet, so configuration and behavior may change between builds. Once it does, changes that break existing configurations will be called out explicitly.

## Why Flow

- **Lower cost, greener.** Flow does more work per CPU and per megabyte of memory, so the same traffic needs less hardware. Fewer and smaller machines mean a smaller bill and a smaller energy footprint, and Flow runs comfortably in modest containers beside your services.
- **Performance.** Every millisecond counts, and in the AI race it counts more. Flow is built for very high throughput with minimal added latency, so the proxy is never the slow part of a request, and streaming model responses reach your users sooner.
- **Security.** Flow is written in Rust, a memory-safe language that rules out whole classes of vulnerabilities by design. On top of that, Flow adds its own layers of protection, so your services are defended from the code that runs the proxy to the traffic that passes through it.
- **Governance.** One place to decide who can reach which service and how much they can use, enforced where the traffic flows. Your whole setup lives in one configuration file you can review, version, and audit, and the same control extends to AI traffic as Flow grows into LLM, MCP, and agent gateways with their own catalogs and registries.
- **Observability.** Metrics, logs, and traces give you the full picture of every request: how many, how fast, what failed, and where the time went.
- **Scalable.** Flow scales up to use every core of a large machine, and scales down to run well on a small one. It also scales out across more instances, so it fits the way your platform already grows. Its tiny size and footprint make it a good fit for serverless as well as for containers and servers, and it is a single executable that starts fast, even from a cold start.
- **Future-ready.** Flow is AI-ready: it is growing into an LLM gateway with a catalog of your models. It is agent-ready, with agent-to-agent traffic and a registry where teams register their agents. It is MCP-ready, with an MCP gateway and a registry where teams register their servers. And it speaks the protocols modern services use: HTTP/2 end to end, gRPC carried natively, and Server-Sent Events streaming.

## What Flow does today

- **Traffic management:** intelligent routing and load balancing across your services.
- **Security:** authentication, access control, and encrypted connections.
- **Resilience:** health monitoring and traffic protection that keep your services available when something fails or demand surges.
- **Performance:** caching and efficient handling of modern and streaming protocols.
- **Observability:** metrics and timing data that show what is happening and where the time goes.
- **Operations:** straightforward configuration and upgrades without downtime.

## Where Flow is heading

Flow is an advanced reverse proxy today. It is growing into:

- **An API gateway**, with more authentication methods, request and response transformation, and further traffic and observability features.
- **An LLM gateway**, with a **model catalog** of the models you have deployed and configured.
- **An MCP gateway**, with an **MCP server registry** where teams register their servers for discovery and governance.
- **An agent-to-agent (A2A) gateway**, with an **agent registry** where teams register their agents for discovery and governance.
- **A skills registry.**

The [Roadmap](roadmap.md) lists what is available and what is planned.

## Get started

1. Get a license file. See [Licensing](license.md).
2. Write a short configuration file. The [Getting Started](getting-started.md) guide walks through one.
3. Run `flow --config-kdl myconfig.kdl`.

## Documentation

### Start here
| Guide | What it covers |
|---|---|
| [Getting started](getting-started.md) | I want to run it: install, run, and verify your first proxy |
| [Technical Overview](technical-overview.md) | How does it work? Core concepts and the request lifecycle |
| [Design Decisions](design-decisions.md) | Which option should I choose? Protocol, load balancing, policies, routing patterns |

### Reference
| Guide | What it covers |
|---|---|
| [Configuration](configuration.md) | Full KDL reference: system, logging, services, listeners, connectors |
| [Command line](cli.md) | Every flag, and how the CLI and config file interact |
| [Routing](routing.md) | Matching requests to routes, and how ties are broken |
| [DNS Resolution](dns.md) | Hostname connectors, background refresh, and what Flow does when DNS misbehaves |
| [Protocols](protocols.md) | HTTP/1.1, HTTP/2, h2c: technical details and performance |
| [Non-REST Protocols](non-rest-protocols.md) | WebSocket, gRPC, SSE, GraphQL, JSON-RPC |
| [Policies](policies.md) | Rate limiting, token authentication, IP restriction, response caching, header rewriting, timing headers, and more |
| [Metrics](metrics.md) | The Prometheus endpoint: every metric, label, and setting (preview) |
| [Performance](performance.md) | Benchmarks, safe operating rates, and tuning |
| [Operations](operations.md) | Running in production, zero-downtime upgrades, logging |
| [Licensing](license.md) | The license file, where Flow looks for it, and what happens at expiration |

### Fit and behavior
| Guide | What it covers |
|---|---|
| [How Flow Behaves](behavior.md) | What to expect when you run Flow: failures, rate limiting across instances, connections, streaming, WebSocket, license check |
| [Where Flow Fits](scope.md) | Is Flow the right tool? Where it sits alongside a service mesh, load balancer, WAF, and cache |
| [FAQ](faq.md) | Short answers with links to the detail |

### Roadmap and changes
| Guide | What it covers |
|---|---|
| [Roadmap](roadmap.md) | What is available, and what is planned |
| [Changelog](changelog.md) | Release notes: what shipped and when, by version |

### Comparisons
| Guide | What it covers |
|---|---|
| [Comparisons](comparisons/README.md) | Measured against other gateways on identical hardware |
