# Non-REST Protocols

> **New here?** Start with [Getting Started](getting-started.md) or [Protocols](protocols.md) for HTTP/1.1, HTTP/2, and h2c.

This page covers Flow's support for protocols and traffic patterns beyond plain REST: WebSocket, gRPC, Server-Sent Events (SSE), GraphQL, JSON-RPC, and similar.

---

## Summary

| Protocol | Status | Notes |
|---|---|---|
| GraphQL | ✅ Works | Plain HTTP POST — no special config |
| JSON-RPC | ✅ Works | Plain HTTP POST — no special config |
| Server-Sent Events (SSE) | ✅ Works | Requires per-route timeout config — see below |
| gRPC (cleartext h2c) | ✅ Works | Requires h2c config |
| gRPCS (TLS) | ✅ Works | Same as any HTTPS + HTTP/2 upstream |
| WebSocket | ⚠️ Passes through | Transparent, not validated — see below |

---

## GraphQL

GraphQL is a query protocol over HTTP. Queries and mutations are HTTP POST requests with a JSON body; subscriptions are typically SSE or WebSocket. Flow proxies GraphQL transparently — no special configuration is required.

**Config**: a standard `PathPrefix` route to your GraphQL endpoint.

```kdl
route {
  match { path-prefix "/graphql" }
  upstream { ... }
}
```

Flow applies policies (auth, rate limiting, headers) to GraphQL requests exactly as it does to any HTTP request. The JSON body is not inspected or parsed.

**GraphQL subscriptions**: if your subscriptions use SSE, see the SSE section below. If they use WebSocket, see the WebSocket section below.

---

## JSON-RPC

JSON-RPC calls are HTTP POST requests with a JSON body. Flow proxies them transparently — no special configuration is required.

```kdl
route {
  match { path-prefix "/rpc" }
  upstream { ... }
}
```

You can route to different backends by path or header if you have multiple JSON-RPC services. Batch requests (arrays in the JSON body) are passed through without modification.

---

## Server-Sent Events (SSE)

SSE is a one-directional streaming protocol over HTTP. The server sends a stream of `text/event-stream` events; the client holds the connection open. Flow proxies SSE over both HTTP/1.1 and HTTP/2.

### Configuration

SSE routes require an extended downstream timeout. Flow's default downstream read timeout is 60 seconds — a client connection silent for more than 60 seconds is closed. For SSE this is too short: an idle event stream (no new events from the server for > 60 s) will be torn down.

Set `disable-read-timeout true` on any route that carries SSE traffic:

```kdl
route {
  match { path-prefix "/events" }
  disable-read-timeout true
  upstream { ... }
}
```

> **Note**: `disable-read-timeout` removes the 60-second cap on the downstream connection. Apply it only to routes that carry long-lived streaming connections, not to general REST routes.

On the upstream side, do not set `read-timeout-secs` on connectors serving SSE or other streaming backends — a per-read timeout will close the upstream connection when the stream is idle.

```kdl
connectors {
  upstream {
    // read-timeout-secs omitted — never set this for streaming upstreams
  }
  "events-backend.internal:8080"
}
```

For HTTP/2 upstreams, `h2-ping-interval-secs` keeps the connection alive without a read timeout:

```kdl
connectors {
  upstream {
    proto "h2-or-h1"
    tls-sni "events-backend.example.com"
    h2-ping-interval-secs 30
  }
  "events-backend.example.com:443"
}
```

### SSE over HTTP/2

SSE works over HTTP/2 — each event stream is a separate H2 stream on a shared connection. This is more efficient than HTTP/1.1 for high fan-out (many clients, one backend). Configure the upstream connector with HTTP/2 as you would for any other service.

---

## gRPC

gRPC runs over HTTP/2. Flow can proxy both gRPC variants:

| Variant | Transport | Config |
|---|---|---|
| `grpc` | HTTP/2 cleartext (h2c) | `proto "h2-only"` + `allow-h2c true` |
| `grpcs` | HTTP/2 over TLS | `proto "h2-only"` + `tls-sni` |

### grpc (cleartext)

```kdl
listeners {
  "0.0.0.0:8000" offer-h2=true   // accept h2c from gRPC clients
}

route {
  match {
    path-prefix "/grpc.YourService"
    header "content-type" "application/grpc"
  }
  upstream "grpc-backend"
}

upstreams {
  upstream "grpc-backend" {
    connectors {
      upstream {
        proto "h2-only"
        allow-h2c true     // cleartext HTTP/2 to the backend
      }
      "grpc-backend.internal:50051"
    }
  }
}
```

The `offer-h2=true` flag on the listener enables h2c detection (HTTP/2 prior knowledge, no TLS). gRPC clients connect directly with HTTP/2 cleartext.

### grpcs (TLS)

```kdl
listeners {
  "0.0.0.0:443" {
    tls {
      certificate-path "/etc/flow/certs/server.crt"
      key-path "/etc/flow/certs/server.key"
    }
  }
}

route {
  match {
    path-prefix "/grpc.YourService"
    header "content-type" "application/grpc"
  }
  upstream "grpcs-backend"
}

upstreams {
  upstream "grpcs-backend" {
    connectors {
      upstream {
        proto "h2-only"
        tls-sni "grpcs-backend.internal"
      }
      "grpcs-backend.internal:50051"
    }
  }
}
```

gRPC TLS is the same as any HTTPS + HTTP/2 upstream. `proto "h2-only"` is stricter than `h2-or-h1` — it fails the request rather than silently downgrading to HTTP/1.1 if the backend does not negotiate HTTP/2 via ALPN.

### Routing gRPC traffic

gRPC uses `POST` with `content-type: application/grpc`. Route by path prefix (the gRPC service name is part of the path) or by the content-type header:

```kdl
// Route by service path
match { path-prefix "/com.example.UserService" }

// Route all gRPC to one backend
match { header "content-type" "application/grpc" }
```

### gRPC streaming

gRPC's four stream patterns (unary, server-streaming, client-streaming, bidirectional) all run over HTTP/2 streams. Flow passes them through without modification. Do not set `read-timeout-secs` or `write-timeout-secs` on connectors serving streaming gRPC — those timeouts apply per read/write and will close streaming calls that pause mid-stream.

### Policies on gRPC

Flow's policies (auth, rate limiting, header manipulation) apply to gRPC requests. One note: gRPC metadata (key/value pairs analogous to HTTP headers) arrives as HTTP/2 headers — the `upsert-header` and `remove-header-key-regex` policies work on them directly. gRPC trailers (trailing metadata sent after the response body) are passed through unmodified; policies do not run on trailers.

---

## WebSocket

WebSocket is a full-duplex protocol established by upgrading an HTTP/1.1 connection (`101 Switching Protocols`).

**Current status**: Flow passes WebSocket upgrade requests through transparently. The underlying mechanism (Pingora's `UpgradedBody` path) handles the protocol switch. However, Flow has not been validated against real WebSocket traffic, and there is no dedicated WebSocket integration test yet.

What this means in practice:

- The HTTP/1.1 upgrade handshake (`GET` + `Connection: Upgrade` + `Upgrade: websocket`) passes through.
- After the 101 response, the connection operates as a raw bidirectional byte stream.
- Flow's request/response policies (`upsert-header`, `rate-limit`, etc.) run on the initial HTTP handshake, not on individual WebSocket frames.
- No subprotocol negotiation, ping/pong injection, or frame-level inspection.

### Required timeout configuration

The same downstream read timeout issue that affects SSE also affects WebSocket. Configure `disable-read-timeout true` on WebSocket routes:

```kdl
route {
  match { path-prefix "/ws" }
  disable-read-timeout true
  upstream { ... }
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
match { path-prefix "/ws" }

// Route by upgrade header
match { header "Upgrade" "websocket" }
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
| Bidirectional real-time (games, chat) | WebSocket — passes through; not validated |
| AI model streaming responses | SSE (most providers) — see timeout note above |

---

## See also

- [Protocols](protocols.md) — HTTP/1.1, HTTP/2, h2c details and benchmarks
- [Configuration Reference](configuration.md) — full KDL syntax for connectors and listeners
- [Limitations](limitations.md) — what Flow does not support
