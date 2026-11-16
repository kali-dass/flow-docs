<p align="center"><img src="images/Flow_light_background_logo.png" alt="Flow Unified Gateway" width="400" height="200"></p>
<h2 align="center">Unified AI Gateway (LLM · MCP · A2A) & API Gateway<br>With a Skills & Agents Registry and a Model Catalogue</h2>
<p align="center"><img src="images/Flow_Diagram.jpg" alt="Flow Unified Gateway Overview" /></p>
<h3>Purpose-built to be hyper-fast, secure, and extremely lightweight</h3>


Flow is a unified gateway for API, LLM, MCP, and A2A traffic — one fast, secure, lightweight proxy instead of a different tool per protocol. It sits in front of your services, handling routing, auth, rate limiting, caching, and observability so your applications don't have to. **Today, Flow is an API gateway; the LLM, MCP, and A2A gateways and the skills/agent registries are next** — see the [Roadmap](roadmap.md).

> **Pre-release software.** Flow has not had a stable release yet, so configuration and behavior may change between builds. Once it does, changes that break existing configurations will be called out explicitly.

## Why Flow

Most platforms end up running a pile of separate tools to carry traffic: an API gateway for REST services, a different gateway in front of LLM calls, an MCP registry to track tool servers, an agent registry for A2A traffic, and yet another catalogue to know which models anyone is even allowed to use. Each one is its own config format, its own auth model, its own blind spot at the seams.

**The bet: one gateway beats five.**

- **Unified.** API, LLM, MCP, A2A — one control plane, not a tool per protocol.
- **Fast and cheap to run.** High throughput, low latency, more work per CPU/MB — fewer machines for the same traffic.
- **Secure by construction.** Rust's memory safety, plus Flow's own policy layers.
- **Governed from one file.** Every access decision reviewable, versionable, auditable in one place.
- **Scales either direction.** A big multi-core box or a small cold-starting container — same binary.

## What Flow does

<details open>
<summary><b>API Gateway</b> — shipped today</summary>

- **Traffic management:** intelligent routing and load balancing across your services.
- **Security:** authentication, access control, and encrypted connections.
- **Resilience:** health monitoring and traffic protection that keep your services available when something fails or demand surges.
- **Performance:** caching and efficient handling of modern and streaming protocols.
- **Observability:** metrics and timing data that show what is happening and where the time goes.
- **Operations:** straightforward configuration and upgrades without downtime.

</details>

<details>
<summary><b>LLM Gateway</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Token-aware rate limiting and spend limits, guardrails, model routing, provider fallover, and AI response caching.

</details>

<details>
<summary><b>Model Catalogue</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Your organization's central list of the models it has purchased, with control over who may use which and a record of how much each team and person has used.

</details>

<details>
<summary><b>MCP Gateway</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Routing, tool-level access control and rate limits, and MCP-aware guardrails.

</details>

<details>
<summary><b>MCP Server Registry</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Where teams register their MCP servers for discovery and governance.

</details>

<details>
<summary><b>A2A Gateway</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Iteration budgets and task correlation across agent delegations.

</details>

<details>
<summary><b>Agents Registry</b> — on the <a href="roadmap.md">Roadmap</a></summary>

Where teams register their agents for discovery and governance.

</details>

<details>
<summary><b>Skills Registry</b> — on the <a href="roadmap.md">Roadmap</a></summary>

A registry for discovering and governing the skills available across your platform.

</details>

The [Roadmap](roadmap.md) lists what is available and what is planned.

## Get started

1. Get a license file. See [Licensing](license.md).
2. Write a short configuration file. The [Getting Started](getting-started.md) guide walks through one — or try this, which routes everything to [httpbin.org](https://httpbin.org/anything):

   ```kdl
   services {
       Demo {
           listeners {
               "0.0.0.0:8080"
           }
           routes {
               route {
                   name "anything"
                   match {
                       path-prefix "/"
                   }
                   allowed-methods "GET"
                   allowed-protocols "http" "h1"
                   connectors {
                       "httpbin.org:443" tls-sni="httpbin.org"
                   }
               }
           }
       }
   }
   ```

3. Point Flow at the license file from step 1 — `export FLOW_LICENSE_FILE=/path/to/license.json` is the simplest way (or pass `--license-file` instead, or drop it in as `./license.json` with no flag or env var at all; see [Licensing](license.md) for the full precedence). Then run `flow --config-kdl myconfig.kdl`, and `curl http://localhost:8080/anything`.

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
| [Metrics](metrics.md) | The Prometheus endpoint: every metric, label, and setting |
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
