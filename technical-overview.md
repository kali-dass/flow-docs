# Technical Overview

> **New here?** Start with [Getting Started](getting-started.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

## What is Flow?

Flow is a high-performance reverse proxy written in Rust. It terminates client connections, routes traffic based on HTTP paths, headers, or TLS SNI, and proxies requests to upstream backends. Flow handles HTTP/1.1 and HTTP/2 on both sides, with optional TLS, and runs everything from a single KDL configuration file.

## How It Works

Request flow through Flow:

```
Client
   |
   | (HTTP/1.1, HTTP/2, or h2c)
   | (with optional TLS)
   ↓
[Listener] ← accepts incoming connections
   |
   | (parsed request)
   ↓
[Router] ← matches request against all routes
   | (picks most specific match by path, header, SNI)
   ↓
[Route] ← contains the destination backend
   | (applies request filters: rate limiting, IP blocking, etc.)
   ↓
[Upstream Selector] ← picks one backend (round-robin, random, hashing)
   |
   | (established connection to backend)
   ↓
[Backend] ← proxies request, receives response
   |
   | (applies response filters: header rewriting, timing, etc.)
   ↓
[Client] ← response sent back
```

## Core Concepts

### Services

A **service** is a single reverse proxy instance that runs within Flow. Each service has its own listeners and routes. You can run multiple services in one Flow process, each handling a different set of clients or listening on different ports.

```kdl
services {
  MyGateway {
    listeners { "0.0.0.0:8080" }
    routes { /* routes here */ }
  }
}
```

### Listeners

A **listener** is a port and protocol combination that accepts client connections.

```kdl
listeners {
  "0.0.0.0:80"          // HTTP on port 80
  "0.0.0.0:443 { tls {} }"  // HTTPS on port 443
  "/tmp/flow.sock"       // Unix domain socket
}
```

Listeners can be plaintext (HTTP) or encrypted (HTTPS/TLS).

### Routes

A **route** specifies how to match a request and where to send it.

```kdl
route {
  match { path-prefix "/api/" }
  connectors {
    "backend1.example.com:80"
    "backend2.example.com:80"
  }
}
```

Every incoming request is matched against all routes. The most specific match wins. A route contains:
- **Matcher** — what requests it handles (path, header, SNI, etc.)
- **Connectors** — which backends to proxy to
- **Policies** — filters to apply (rate limiting, IP blocking, header rewriting)

### Connectors (Upstreams)

A **connector** is an upstream backend server. When a route matches, Flow picks one connector (backend) and proxies the request there.

```kdl
connectors {
  load-balance { selection "RoundRobin" }
  "backend1.example.com:80"
  "backend2.example.com:80"
}
```

Connectors can be HTTP/1.1, HTTP/2, or h2c, with optional TLS.

### Load Balancing

When a route has multiple connectors, Flow picks one using the **load balancing strategy**:

- **Round-robin**: Cycle through backends in order
- **Random**: Pick randomly
- **Consistent hashing**: Hash request properties (IP, path) to always route to the same backend

```kdl
load-balance { selection "RoundRobin" }
```

### Policies

**Policies** are filters that run before or after proxying:

- **Rate limiting**: Max requests per second (global, per-IP, or per-path)
- **IP blocking**: Block requests from CIDR ranges
- **Request header rewriting**: Add, remove, or modify headers
- **Response header rewriting**: Inject or strip response headers
- **Timing headers**: Add latency measurements for debugging

```kdl
path-control {
  block-cidr-range { ranges "192.168.0.0/16" }
}

upstream-request {
  upsert-header { key "X-Forwarded-By" value "Flow" }
}

upstream-response {
  timing-header {}
}
```

## Request Lifecycle

Every request follows the same path:

1. **Accept** — Listener accepts connection from client (with optional TLS handshake)
2. **Parse** — Flow parses HTTP headers
3. **Route** — Find the most specific matching route
4. **Policy: Pre-upstream** — Apply request-phase policies (rate limiting, IP blocking)
5. **Select backend** — Choose one connector using the load balancing strategy
6. **Connect upstream** — Establish connection to backend (with optional TLS)
7. **Forward request** — Send request to backend, receive response
8. **Policy: Post-upstream** — Apply response-phase policies (header rewriting, timing)
9. **Respond to client** — Send response back to client
10. **Close or reuse** — Close connection or keep alive for next request

All steps use the configuration from the matching route. If multiple steps require the same information (e.g., request headers), it's parsed once and reused.

## Key Design Choices

### Why Routing by Path, Header, and SNI?

These three dimensions cover the most common routing use cases:

- **Path** — Route to different services based on URL path (e.g., `/api/` → API service, `/static/` → CDN)
- **Header** — Route based on tenant ID, API key, or feature flag (multi-tenancy without separate ports)
- **SNI** — Virtual hosting at the TLS layer (e.g., `api.example.com` vs `admin.example.com` both on port 443)

Together, they handle 90% of routing needs without requiring request body inspection or other expensive operations.

### Why Terminate TLS at the Gateway?

- **Centralized certificate management** — One place to manage certificates and rotations
- **Offload crypto from backends** — Backends stay fast, don't do TLS handshakes
- **Visibility** — Can inspect headers (for routing) only after TLS termination
- **Flexibility** — Re-encrypt differently to each backend (TLS, h2c, etc.)

### Why h2c for Internal Networks?

HTTP/2 multiplexing (multiple streams per connection) is powerful but TLS handshakes are expensive. For trusted internal networks (Kubernetes clusters, internal services):

- **Use h2c** (HTTP/2 cleartext) to skip TLS overhead
- **Gain 2× throughput** (h2c both hops) vs HTTP/1.1
- **Lower latency** (fewer connections = less queueing)
- **Simpler infrastructure** (no certs needed internally)

## Next Steps

- **Getting started?** See [Getting Started](getting-started.md) for your first Flow config
- **Choosing architecture?** See [Design Decisions](design-decisions.md) for protocol/load-balancing/policy guidance
- **Understanding protocols?** See [Protocols](protocols.md) for HTTP/1.1, HTTP/2, and h2c details
- **Understanding routing?** See [Routing](routing.md) for matching rules and scoring
- **Configuration syntax?** See [Configuration Reference](configuration.md) for complete KDL documentation
- **Available policies?** See [Policies](policies.md) for all filter types
- **Performance and tuning?** See [Performance](performance.md) for benchmarks and capacity planning
- **Production deployment?** See [Operations](operations.md) for zero-downtime upgrades, logging, and monitoring
- **Understanding constraints?** See [Limitations](limitations.md) for what Flow can't do yet and known behaviors
