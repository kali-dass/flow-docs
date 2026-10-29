# Non-REST Protocols

> **New here?** Start with [Getting Started](getting-started.md) or [Protocols](protocols.md) for HTTP/1.1, HTTP/2, and h2c.

This page covers Flow's support for protocols and traffic patterns beyond plain REST: WebSocket, gRPC, Server-Sent Events (SSE), GraphQL, JSON-RPC, and similar.

---

## Summary

| Protocol | Status | Notes |
|---|---|---|
| GraphQL | ✅ Works | Plain HTTP POST — no special config |
| JSON-RPC | ✅ Works | Plain HTTP POST — no special config |
| Server-Sent Events (SSE) | ✅ Works | No downstream timeout needed; see below |
| gRPC (cleartext h2c) | ✅ Works | Plaintext listener with `offer-h2=true`; route `allowed-protocols "grpc"`; connector `proto="grpc"` |
| gRPCS (TLS from clients) | ✅ Works | TLS listener with `offer-h2=true`; route `allowed-protocols "grpcs"` |
| gRPCS (TLS to the backend) | ✅ Works | Connector `proto="grpcs"` with `tls-sni` |
| WebSocket | ⚠️ Passes through | Transparent; see below and the [Roadmap](roadmap.md#protocols) |

---

## GraphQL

GraphQL is a query protocol over HTTP. Queries and mutations are HTTP POST requests with a JSON body; subscriptions are typically SSE or WebSocket. Flow proxies GraphQL transparently — no special configuration is required.

**Config**: a standard `PathPrefix` route to your GraphQL endpoint.

```kdl
route {
  name "graphql"
  match {
      path-prefix "/graphql"
  }
  connectors {
    "graphql-backend.internal:4000"
  }
}
```

Flow applies policies (auth, rate limiting, headers) to GraphQL requests exactly as it does to any HTTP request. The JSON body is not inspected or parsed.

**GraphQL subscriptions**: if your subscriptions use SSE, see the SSE section below. If they use WebSocket, see the WebSocket section below.

---

## JSON-RPC

JSON-RPC calls are HTTP POST requests with a JSON body. Flow proxies them transparently — no special configuration is required.

```kdl
route {
  name "rpc"
  match {
      path-prefix "/rpc"
  }
  connectors {
    "jsonrpc-backend.internal:8080"
  }
}
```

You can route to different backends by path or header if you have multiple JSON-RPC services. Batch requests (arrays in the JSON body) are passed through without modification.

---

## Server-Sent Events (SSE)

SSE is a one-directional streaming protocol over HTTP. The server sends a stream of `text/event-stream` events; the client holds the connection open. Flow proxies SSE over both HTTP/1.1 and HTTP/2.

### Configuration

An SSE route needs no special downstream setting. Flow's default 60-second downstream read timeout applies to the client's request, and a stream that is silent for longer than that stays open: the client has sent nothing, so there is nothing to time out. An SSE response to a plain `GET` and an SSE response to a `POST` with a body (the pattern used by LLM streaming APIs) both behave this way.

```kdl
route {
    name "events"
    match {
        path-prefix "/events"
    }
    allowed-methods "GET"
    allowed-protocols "http" "https" "h1" "h2"
    connectors {
        "10.0.0.6:8080"
    }
}
```

To reclaim connections from HTTP/1.1 clients that stop consuming events, add `downstream-write-timeout`. It caps how long Flow waits for a client to accept a write, applies only to HTTP/1.1 clients, and is off by default:

```kdl
route {
    name "events"
    match {
        path-prefix "/events"
    }
    downstream-write-timeout 30     // drop clients that stall for more than 30 s on a write
    allowed-methods "GET"
    allowed-protocols "http" "https" "h1" "h2"
    connectors {
        "10.0.0.6:8080"
    }
}
```

For every timeout by traffic type, with complete examples, see [Choosing timeouts by traffic type](configuration.md#choosing-timeouts-by-traffic-type).

### Upstream timeout (applies to both H1 and H2)

Do not set `read-timeout-secs` or `write-timeout-secs` on connectors serving SSE or other streaming backends. Unlike `downstream-read-timeout` and `downstream-write-timeout`, these connector timeouts apply per read/write operation on **both HTTP/1.1 and HTTP/2 upstream connections** — they will close the upstream connection whenever the stream is idle between events, regardless of protocol.

```kdl
connectors {
    // read-timeout-secs and write-timeout-secs omitted: never set these for streaming upstreams
    "10.0.0.6:8080"
}
```

For HTTP/2 upstreams, `h2-ping-interval-secs` keeps the connection alive at the protocol level without needing a read timeout:

```kdl
connectors {
    "10.0.0.6:8443" tls-sni="events-backend.example.com" proto="h2-or-h1" h2-ping-interval-secs=30
}
```

### SSE over HTTP/2

SSE works over HTTP/2 — each event stream is a separate H2 stream on a shared connection. This is more efficient than HTTP/1.1 for high fan-out (many clients, one backend). Configure the upstream connector with HTTP/2 as you would for any other service.

---

## gRPC

gRPC runs over HTTP/2. Flow can proxy both gRPC variants:

| Variant | Transport | Config |
|---|---|---|
| `grpc` | HTTP/2 cleartext (h2c) | `proto="grpc"` (same as `proto="h2-only"` + `allow-h2c=true`) |
| `grpcs` | HTTP/2 over TLS | `proto="grpcs"` + `tls-sni` (same as `proto="h2-only"` + `tls-sni`) |

The two hops are independent, so all four combinations work:

| Client to Flow | Flow to backend | Listener | Route | Connector |
|---|---|---|---|---|
| Cleartext | Cleartext | `offer-h2=true` | `allowed-protocols "grpc"` | `proto="grpc"` |
| TLS | Cleartext | `cert-path`, `key-path`, `offer-h2=true` | `allowed-protocols "grpcs"` | `proto="grpc"` |
| Cleartext | TLS | `offer-h2=true` | `allowed-protocols "grpc"` | `proto="grpcs" tls-sni="<name>"` |
| TLS | TLS | `cert-path`, `key-path`, `offer-h2=true` | `allowed-protocols "grpcs"` | `proto="grpcs" tls-sni="<name>"` |

Flow does not detect gRPC. You declare it twice, and each declaration fixes one hop:

- **On the route**, `allowed-protocols "grpc"` (clients on cleartext HTTP/2) or `"grpcs"` (clients on TLS). List both to accept either. It implies `POST` and HTTP/2, so you no longer write `allowed-methods "POST"` or `allowed-protocols "h2"`, and Flow refuses a client on the wrong transport.
- **On the connector**, `proto="grpc"` (cleartext to the backend) or `proto="grpcs"` (TLS to the backend, with `tls-sni`). A gRPC route needs every connector to be HTTP/2 only.

The longer `proto="h2-only"` spellings are equivalent for connectors.

### grpc (cleartext)

```kdl
listeners {
  "0.0.0.0:8000" offer-h2=true   // accept h2c from gRPC clients
}

route {
  name "grpc-yourservice"
  match {
    path-prefix "/grpc.YourService"
    header name="content-type" value-regex="^application/grpc"
  }
  allowed-protocols "grpc"
  connectors {
    load-balance {
        selection "RoundRobin"
    }
    "grpc-backend.internal:50051" proto="grpc"
  }
}
```

The `offer-h2=true` attribute on the listener enables h2c detection (HTTP/2 prior knowledge, no TLS). gRPC clients connect directly with HTTP/2 cleartext.

### grpcs (TLS)

gRPC over TLS has two hops, and each can be encrypted independently.

#### TLS from clients to Flow

The common setup. Flow terminates TLS from the gRPC clients and forwards cleartext HTTP/2 to the backend, so the backend needs no certificate. The listener carries the certificate and key and negotiates HTTP/2 through ALPN:

```kdl
services {
    Grpc {
        listeners {
            "0.0.0.0:8443" cert-path="/etc/flow/certs/server.crt" key-path="/etc/flow/certs/server.key" offer-h2=true
        }
        routes {
            route {
                name "routeguide"
                match {
                    path-prefix "/routeguide.RouteGuide"
                }
                allowed-protocols "grpcs"
                connectors {
                    "10.0.0.8:10000" proto="grpc"
                }
            }
        }
    }
}
```

Clients connect to an `https://` address and trust the listener's certificate. A client that dials by name must use a name the certificate covers. A service cannot mix this TLS listener with a plaintext `offer-h2` listener.

#### TLS from Flow to the backend

When the network between Flow and the backend is not trusted, encrypt that hop too. The connector uses `proto="grpcs"` with `tls-sni`, instead of `proto="grpc"`. The client side is independent: the example below takes cleartext gRPC from clients, and a TLS listener (see above) works the same way.

```kdl
services {
    Grpc {
        listeners {
            "0.0.0.0:8080" offer-h2=true
        }
        routes {
            route {
                name "grpc-yourservice"
                match {
                    path-prefix "/grpc.YourService"
                }
                allowed-protocols "grpc"
                connectors {
                    "grpcs-backend.internal:50051" proto="grpcs" tls-sni="grpcs-backend.internal" ca-cert-path="/etc/flow/backend-ca.crt"
                }
            }
        }
    }
}
```

`tls-sni` is the name Flow checks against the backend's certificate. With a certificate signed by a CA, set `ca-cert-path` to that CA's certificate file, or omit it if the CA is in the system trust store.

`proto="h2-only"` is stricter than `h2-or-h1`: if the backend does not negotiate HTTP/2 through ALPN, Flow returns 502 rather than silently downgrading to HTTP/1.1.

**Self-signed backend certificates.** Flow does not accept a self-signed leaf certificate as a trust anchor in `ca-cert-path`, and the connection fails with a certificate verification error. For a verified hop, use a certificate signed by a CA, and point `ca-cert-path` at that CA. For development with a self-signed certificate, set `verify-cert=false` on the connector, which turns certificate verification off for that backend and logs a warning at startup. Never use it in production.

### Routing gRPC traffic

gRPC uses `POST` with `content-type: application/grpc`. Route by path prefix (the gRPC service name is part of the path) or by the content-type header:

The regex matches `application/grpc` and variants such as `application/grpc+proto`. Several matchers in one `match` block must all match (see [Routing](routing.md#implicit-and--multiple-matchers-in-one-match-block)).

```kdl
// Route by service path
match {
    path-prefix "/com.example.UserService"
}

// Route all gRPC to one backend
match {
    header name="content-type" value-regex="^application/grpc"
}
```

### gRPC streaming

gRPC's four stream patterns (unary, server-streaming, client-streaming, bidirectional) all run over HTTP/2 streams. Flow passes them through without modification.

**No downstream timeout configuration needed.** `downstream-read-timeout` has no effect on HTTP/2 connections — the 60-second read timeout only applies to HTTP/1.1 downstream connections. gRPC clients always connect over H2 (h2c or TLS), so long-lived streaming RPCs are never closed by that timer.

Do not set `read-timeout-secs` or `write-timeout-secs` on connectors serving streaming gRPC — those timeouts apply per read/write and will close streaming calls that pause mid-stream.

Timeouts belong to the connector, so when one service mixes call types, give each method its own route. A route that serves only unary calls can set `read-timeout-secs` to catch a hung backend, while the routes for streaming methods leave it unset and use `h2-ping-interval-secs` to detect a dead connection:

```kdl
route {
    name "orders-get"
    match {
        path-prefix "/shop.Orders/GetOrder"
    }
    allowed-protocols "grpc"
    connectors {
        "[::1]:50051" proto="grpc" read-timeout-secs=10 write-timeout-secs=10
    }
}

route {
    name "orders-watch"
    match {
        path-prefix "/shop.Orders/WatchOrders"
    }
    allowed-protocols "grpc"
    connectors {
        "[::1]:50051" proto="grpc" h2-ping-interval-secs=30
    }
}
```

The routes share one connection to the backend, and each keeps its own timeouts.

### What has been validated

Tested end to end with a Rust gRPC client and server (the RouteGuide example, which has all four call types):

- Unary, server-streaming, client-streaming and bidirectional-streaming calls through all four client and backend combinations in the table above.
- Bidirectional streams that stay open for over 60 seconds.
- A separate route and timeouts for each call type, with several routes sharing one backend connection.
- Flow's own errors reaching a gRPC client as gRPC statuses: no route, backend unreachable, rate limit, IP restriction, request termination, and a client on the wrong transport (see [Errors Flow produces on a gRPC route](#errors-flow-produces-on-a-grpc-route)).

Not yet validated: gRPC clients and servers written with other libraries (Go, Java), very large messages, client deadlines (`grpc-timeout`) and cancellation, and a write timeout on a server-streaming route under a slow client. Test these in your environment before relying on them. See [Roadmap](roadmap.md#protocols) for status.

### Errors Flow produces on a gRPC route

Responses from your backend always pass through unchanged. When **Flow itself** answers instead (it rejected the call, or could not reach the backend), a plain HTTP error would leave a gRPC client guessing a status from the HTTP code. On a route declared with `allowed-protocols "grpc"` or `"grpcs"`, Flow answers in gRPC: HTTP `200`, `content-type: application/grpc`, a `grpc-status` and a `grpc-message`, and no body.

| What happened | gRPC status | `grpc-message` |
|---|---|---|
| No route matches the call | `UNIMPLEMENTED` | No configured route matches this request |
| Backend unreachable, or it did not answer in time | `UNAVAILABLE` | Upstream unavailable |
| Rate limit exceeded | `RESOURCE_EXHAUSTED` | Too many requests |
| Token missing or invalid | `UNAUTHENTICATED` | the policy's message |
| Authentication service unavailable | `UNAVAILABLE` | authentication service unavailable |
| IP restriction | `PERMISSION_DENIED` | your rejection message, or Access denied |
| `request-termination` with an error status | from the status you set | the status's standard text |
| Client on the wrong transport for the route | `FAILED_PRECONDITION` | This route requires a secure (TLS) connection, or accepts gRPC over cleartext HTTP/2 only |

Notes:

- **Rate limits are `RESOURCE_EXHAUSTED`, not `UNAVAILABLE`.** Clients retry `UNAVAILABLE`, which is wrong for a limit.
- **No route is the one case without a declaration to consult.** There Flow looks at the request: an HTTP/2 call with a `content-type` of `application/grpc` gets the gRPC form. Any other request gets the usual JSON error.
- **HTTP/1.1 clients keep the HTTP error.** A gRPC response needs HTTP/2, so an HTTP/1.1 request to a gRPC route still gets `400`.
- **Other routes are unchanged.** Routes that are not declared `grpc` or `grpcs` keep their JSON errors and HTTP status codes.
- **In metrics** these responses are HTTP 200, so they count as `status_class="2xx"`. Filter on `source="flow"` to tell them apart from calls your backend answered.

### Policies on gRPC

Flow's policies (auth, rate limiting, header manipulation) apply to gRPC requests. One note: gRPC metadata (key/value pairs analogous to HTTP headers) arrives as HTTP/2 headers — the `upsert-header` and `remove-header-key-regex` policies work on them directly. gRPC trailers (trailing metadata sent after the response body) are passed through unmodified; policies do not run on trailers.

---

## WebSocket

WebSocket is a full-duplex protocol established by upgrading an HTTP/1.1 connection (`101 Switching Protocols`).

Flow passes WebSocket upgrade requests through transparently. The proxy's upgrade handling performs the protocol switch.

> **Preview:** WebSocket passthrough. See [Roadmap](roadmap.md#protocols).

What this means in practice:

- The HTTP/1.1 upgrade handshake (`GET` + `Connection: Upgrade` + `Upgrade: websocket`) passes through.
- After the 101 response, the connection operates as a raw bidirectional byte stream.
- Flow's request/response policies (`upsert-header`, `rate-limit`, etc.) run on the initial HTTP handshake, not on individual WebSocket frames.
- No subprotocol negotiation, ping/pong injection, or frame-level inspection.

### Required timeout configuration

After the `101` upgrade, Flow keeps reading the client side of the connection, and the default 60-second downstream read timeout closes a WebSocket that is silent for longer. Configure `downstream-read-timeout "None"` on WebSocket routes:

```kdl
route {
  name "ws"
  match {
      path-prefix "/ws"
  }
  downstream-read-timeout "None"
  allowed-methods "GET"
  allowed-protocols "http" "https" "h1" "h2"
  connectors {
      "10.0.0.7:8080"
  }
}
```

Without this, connections silent for more than 60 seconds are closed — WebSocket keepalive pings are not enough if the client is silent between pings.

Also do not set `read-timeout-secs` or `write-timeout-secs` on upstream connectors serving WebSocket backends.

### WebSocket over HTTP/2 (RFC 8441)

Not currently supported. WebSocket-over-HTTP/2 extended CONNECT is a different mechanism from HTTP/1.1 Upgrade and is not wired up. Use HTTP/1.1 for WebSocket traffic.

### Routing WebSocket traffic

Route WebSocket upgrade requests by path or by the `Upgrade` header:

```kdl
// Route by path
match {
    path-prefix "/ws"
}

// Route by upgrade header
match {
    header name="Upgrade" value-regex="(?i)^websocket$"
}
```

---

## Protocol choice guide

| Use case | Recommended protocol |
|---|---|
| REST APIs, web backends | HTTP/1.1 or HTTP/2 (see [Protocols](protocols.md)) |
| GraphQL queries and mutations | HTTP/1.1 or HTTP/2 — transparent |
| GraphQL subscriptions | SSE (preferred) or WebSocket |
| JSON-RPC (single or batch) | HTTP/1.1 or HTTP/2 — transparent |
| Push notifications, live feeds | SSE |
| Microservice RPC (internal) | gRPC (cleartext h2c) |
| Microservice RPC (external or cross-region) | gRPCS (TLS) |
| Bidirectional real-time (games, chat) | WebSocket — passes through (see [Roadmap](roadmap.md#protocols)) |
| AI model streaming responses | SSE (most providers) — see timeout note above |

---

## See also

- [Protocols](protocols.md) — HTTP/1.1, HTTP/2, h2c details and benchmarks
- [Configuration Reference](configuration.md) — full KDL syntax for connectors and listeners
- [How Flow Behaves](behavior.md) — failures, streaming, and WebSocket behavior
