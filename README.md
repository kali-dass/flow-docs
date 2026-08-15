# Flow documentation

**Flow** is a high-performance AI and API Gateway built in Rust. It routes HTTP/1.1 and HTTP/2
traffic to upstream services based on paths, headers, and SNI, with load balancing, TLS
termination, rate limiting, and header rewriting — all driven by a single KDL config file.

## Start here

| Guide | What it covers |
|---|---|
| [Getting started](getting-started.md) | Install, run, and verify your first proxy |
| [Command line](cli.md) | Every flag, and how the CLI and config file interact |
| [Configuration](configuration.md) | Full KDL reference — system, services, listeners, connectors |
| [Routing](routing.md) | Matching requests to routes, and how ties are broken |
| [Policies](policies.md) | Rate limiting, IP blocking, header rewriting, timing |
| [Performance](performance.md) | Benchmarks, safe operating rates, and tuning |
| [Operations](operations.md) | Running in production, zero-downtime upgrades, logging |
| [Comparisons](comparisons/README.md) | Measured against other gateways on identical hardware |

## At a glance

```kdl
services {
    MyGateway {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                match { path-prefix "/api" }
                allowed-methods "GET" "POST"
                allowed-protocols "http"
                connectors {
                    load-balance { selection "RoundRobin" }
                    "10.0.0.1:3000"
                    "10.0.0.2:3000"
                }
            }
        }
    }
}
```

```bash
flow --config-kdl config.kdl
```

That's a working load-balanced proxy: requests to `/api*` on port 8080 are round-robined
across two backends.

## What Flow is good at

- **Speed.** CPU-bound, not I/O-bound — throughput scales with cores and depends heavily on
  your configuration (TLS vs. cleartext, HTTP/1.1 vs. HTTP/2), ranging from ~14,300 to
  ~28,000 requests/second on 2 CPU cores in testing. See [Performance](performance.md) for
  the full measurements, what drives the range, and how to reproduce them.
- **Flexible routing.** Match on path prefix, path regex, header values, or TLS SNI —
  and combine them with AND/OR logic.
- **TLS everywhere.** Terminate TLS from clients, originate TLS to upstreams, or run
  cleartext HTTP/2 (h2c) to trusted internal backends for maximum throughput.
- **Zero-downtime reloads.** Hand listening sockets to a new process without dropping a
  connection.

## What Flow does not do (yet)

- WebSocket proxying
- gRPC support (implemented via existing H2/streaming primitives, not yet validated against real gRPC traffic)
- upstream health checking
