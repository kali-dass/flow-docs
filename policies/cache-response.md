# Cache response

Caches upstream responses in memory and serves subsequent matching requests directly from the
cache, bypassing the backend entirely.

**Configuration:** `cache-response` is a route-level block — not a `path-control` filter.

```kdl
routes {
    route {
        name "api-v1"
        match {
            path-prefix "/api/v1"
        }
        upstream "backend"
        cache-response {
            ttl-secs 60
            max-body-bytes 1048576
            max-capacity-bytes 104857600
            cache-status-codes 200 201        // multiple status codes: space-separated arguments
            allowed-methods "GET"             // multiple methods: space-separated string arguments (HEAD is opt-in)
            key-headers "Accept-Language"     // multiple headers: space-separated string arguments
            identity-from "none"
        }
    }
}
```

All keys are optional. The example above shows explicit values; omitting a key uses its default.

## Options

| Key | Default | Description |
|---|---|---|
| `ttl-secs` | `60` | How long a cached entry lives, in seconds |
| `max-body-bytes` | `1048576` (1 MiB) | Maximum response body size to cache; larger responses pass through uncached |
| `max-capacity-bytes` | `104857600` (100 MiB) | Maximum total memory used by the cache store for this route |
| `cache-status-codes` | `200` | One or more HTTP status codes eligible for caching, written as separate integer arguments: `cache-status-codes 200 201 204`. `1xx`, `206` and `304` cannot be listed; see [Status codes that cannot be cached](#status-codes-that-cannot-be-cached) |
| `allowed-methods` | `"GET"` | One or more HTTP methods eligible for cache lookup and storage, written as separate string arguments: `allowed-methods "GET" "HEAD"`. `HEAD` is opt-in; see [Responses to think about](#responses-to-think-about-before-caching) |
| `key-headers` | _(none)_ | One or more request header names to include in the cache key, written as separate string arguments: `key-headers "Accept-Language" "X-Tenant-Id"` |
| `identity-from` | `"none"` | Scope the cache key to a per-identity namespace (see [Identity isolation](#identity-isolation)) |
| `cache-content-types` | _(none — allow all)_ | Allowlist of content-type prefixes to cache. When set, only responses whose `Content-Type` starts with one of the listed values are stored. Values are matched case-insensitively before any `;` parameter: `cache-content-types "application/json" "text/plain"` |

## Cache status

Every response from a cached route includes `X-Cache-Status`:

| Value | Meaning |
|---|---|
| `HIT` | Response served from cache — backend was not contacted |
| `MISS` | Response from backend — now stored in cache |
| `BYPASS` | Cache bypassed for this request (see [Bypass conditions](#bypass-conditions)) |

A `HIT` carries a `Content-Length` for the stored body (whether or not the backend sent one, since a backend may stream a response in chunks), so a client can reuse its connection after a hit. An `Age` header is injected on `HIT`, indicating how many seconds ago Flow received this
response from the upstream. On `MISS` and `BYPASS` there is no `Age` header — any `Age` the
upstream sent is stripped before storage (see [Headers stripped before storage](#headers-stripped-before-storage)).

## Cache key

The default cache key is `<METHOD> <path><query>`. When `key-headers` are configured, the
values of those headers are appended in declaration order.

### Vary

When the upstream response includes a `Vary` header, Flow automatically includes the values of
the named request headers in the cache key. Each distinct combination of Vary header values is
stored as a separate entry. The variant index is bounded at 64 entries per URL; older variants
are evicted first.

`Vary: *` means the response is never reusable — Flow does not cache these responses.

## Bypass conditions

Flow bypasses the cache (`X-Cache-Status: BYPASS`) for a request when:

- The HTTP method is not in `allowed-methods`
- The client sends `Cache-Control: no-cache` or `Cache-Control: no-store`
- The `identity-from` source is absent (see [Identity isolation](#identity-isolation))

When the client sends `Cache-Control: only-if-cached` (RFC7234 §5.4), Flow returns `504 Gateway Timeout` on a MISS rather than forwarding to the upstream. The client explicitly requested a cached copy and is not willing to wait for a fresh fetch.

Flow does not **store** a response when:

- The response status code is not in `cache-status-codes`
- The response body is larger than `max-body-bytes`
- The response includes `Set-Cookie`
- The upstream response includes `Cache-Control: private`, `no-store`, or `no-cache`
- The response carries `Vary: *`
- The response `Content-Type` is not in `cache-content-types` (when that option is set)

## Responses to think about before caching

`cache-response` is a choice you make for a route. Flow stores any response that passes the rules above; it does not look at what the response is for. A useful test before you turn it on: would two different callers be happy to receive the same bytes for `ttl-secs`? Some kinds of response need care:

| Response | What happens when it is cached | What to do |
|---|---|---|
| **Live streams: Server-Sent Events** (`text/event-stream`) | A server-sent stream that ends within `max-body-bytes` is stored, and a hit replays every stored event at once, so the live timing is lost. A browser's `EventSource` reconnects when a stream ends, so it receives the same stored events again on every reconnect until the entry expires. `Last-Event-ID` is not part of the cache key, so a client resuming after event 40 gets the copy from event 1. Your backend never sees the clients served from the cache. A backend that sends `Cache-Control: no-cache` is never stored. | Do not put `cache-response` on a route that serves live streams. On a route that mixes streams with ordinary responses, allow only the types you want: `cache-content-types "application/json"`. |
| **Newline-delimited JSON** (NDJSON, JSON Lines) | Cached like any other body. A finished export replays correctly. A feed whose lines change between calls is replayed as the stored snapshot until the entry expires. | Cache routes whose answer is the same for `ttl-secs`. Leave live feeds uncached. |
| **Per-user or per-tenant responses** | Without `identity-from`, the first caller's response is served to everyone. | Set [`identity-from`](#identity-isolation), and put [token authentication](token-auth.md) on the route. |
| **Responses that differ by request header** (language, tenant, format) | A `Vary` from the backend is honored. Without one, callers share one entry. | Add the header to `key-headers`, or have the backend send `Vary`. |
| **Methods other than `GET`** | The cache key does not include the request body, so different `POST` bodies to one URL share an entry and every caller gets the first answer. | Keep the default `allowed-methods`. Do not add `POST` for an API whose answer depends on the body, such as search or an LLM completion. |
| **`HEAD`** (opt-in: `allowed-methods "GET" "HEAD"`) | A `HEAD` entry is stored separately from the `GET` entry for the same URL. Within `ttl-secs` the two can disagree: if the backend changes a resource after the `GET` was cached, a `HEAD` that misses reports the new size or `ETag` while a `GET` hit still returns the old body. A client that checks the size with `HEAD` and then downloads with `GET` sees the mismatch. A `HEAD` hit keeps the origin's `Content-Length`. | Leave `HEAD` out of `allowed-methods` (the default), so a `HEAD` always reaches the backend and is `BYPASS`. List it only when the disagreement within `ttl-secs` is acceptable. |
| **`204 No Content`** (only if listed in `cache-status-codes`) | A `204` is stored like any other response and repeated for `ttl-secs`. A polling endpoint that answers `204` for "nothing new" keeps answering "nothing new" for up to `ttl-secs`, even after the backend has something. | Do not list `204` for polling endpoints, or lower `ttl-secs`. |
| **gRPC** | The key ignores the request message and the trailers that carry `grpc-status` are not stored, so every call would receive the first response. | Flow refuses to start with `cache-response` on a route declared `allowed-protocols "grpc"` or `"grpcs"`, however the block is configured. |
| **Large or open-ended responses** | A response larger than `max-body-bytes` passes through uncached every time. While a response is arriving, a miss holds up to `max-body-bytes` in memory. | Set `max-body-bytes` for what you want cached, not for the largest response you serve. |
| **Fast-changing data** (prices, status, counters) | Callers see data up to `ttl-secs` old. This includes health, readiness and status endpoints that sit under a cached route prefix: a probe that goes through such a route sees the cached result, not the backend's current state. | Lower `ttl-secs`, or leave the route uncached. Give an endpoint that reports current state its own route without `cache-response`. Flow's own [active health checks](health-checking.md) probe each backend directly and never go through routes or the cache, so they are not affected. |

If a route needs both cached and uncached responses, use `cache-content-types` to name the types to store. A response over a [Response size limit](response-size-limit.md) is never stored, so a hit is always served from a response that passed the limit.

## Status codes that cannot be cached

Flow refuses to start when `cache-status-codes` lists any of these, with a message naming the code and the reason:

| Status | Why it cannot be cached |
|---|---|
| `1xx` (`100`, `101`, ... `199`) | An informational response is never the final response to a request. |
| `206 Partial Content` | It is a part of a resource, and the cache key has no `Range`, so the part would be replayed to a request for the whole resource. |
| `304 Not Modified` | It answers a conditional request ("your copy is still good") and is not a representation of the resource. Replayed to a client that sent no condition, it would be a response with no body that client cannot use. |

A response with no body is stored like any other: a `204` (when listed), a `200` with `Content-Length: 0`, and a `HEAD` response (when `HEAD` is in `allowed-methods`).

## Headers stripped before storage

Hop-by-hop headers are stripped and never replayed: `Transfer-Encoding`, `Connection`,
`Keep-Alive`, `Upgrade`, `Via`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`,
and `Age`.

Rate-limit headers are also stripped — any header whose name begins with `RateLimit-`,
`X-RateLimit-`, or `X-Rate-Limit-` reflects a per-connection, per-time-window value and would
be misleading if replayed from a cached response.

## Order of processing

The cache is consulted **after** the request filters: a request is first checked by the service-level and route-level filters (authentication, IP restriction, rate limits, request size) and only a request that passes can be answered from the cache. A cached response is therefore never served to a caller that [Token authentication](token-auth.md), [IP restriction](ip-restriction.md) or a rate limit would have refused, and every hit counts against the rate limit.

## Identity isolation

By default all requests share the same cache key space. Use `identity-from` to scope the cache
key to a per-identity namespace — required when different authenticated users receive different
upstream responses. The identity comes from [Token authentication](token-auth.md) on the same route, which runs before the cache is consulted. Flow refuses to start when `identity-from` needs an identity (`client-id`, `tenant-id` or a claim) and the route has no `token-auth` in its request filters, at either the service or the route level.

| `identity-from` value | Identity source |
|---|---|
| `"none"` (default) | Shared key space — no isolation |
| `"client-id"` | `ctx.identity.client_id` — set by `token-auth` via `client-id-claim` |
| `"tenant-id"` | `ctx.identity.tenant_id` — set by `token-auth` via `tenant-id-claim` |
| `"header:<name>"` | Value of a named request header (e.g. `"header:x-consumer-id"`) |
| `"claim:<path>"` | A JWT claim value by literal name or JSON Pointer path (e.g. `"claim:sub"` or `"claim:/org/id"`) |

When `identity-from` is anything other than `"none"` and the identity source cannot be resolved,
the request BYPASSes the cache. Flow logs a warning on the first occurrence.

### Nested JWT claims

`claim:<path>` supports JSON Pointer paths (RFC 6901) for nested claims:

| Syntax | Meaning |
|---|---|
| `"claim:sub"` | Flat literal key `claims["sub"]` |
| `"claim:https://myapp.com/org_id"` | Flat literal key — URI-style names containing `/` are literals, not pointers |
| `"claim:/org/id"` | One level deep: `claims["org"]["id"]` |
| `"claim:/context/tenant/id"` | Two levels deep: `claims["context"]["tenant"]["id"]` |
| `"claim:/org~1division"` | Key with `/` in name: `claims["org/division"]` (`~1` → `/`) |
| `"claim:/user~0role"` | Key with `~` in name: `claims["user~role"]` (`~0` → `~`) |
| `"claim:/a~01b"` | Key with literal `~1` in name: `claims["a~1b"]` — `~1` applied before `~0` (RFC 6901 §3) |

`claim:` requires `expose-claims true` on the `token-auth` plugin. Without it `raw_claims` is
not populated and every request BYPASSes.

### Tenant vs user isolation

- `"tenant-id"` — all users in the same tenant share a cache namespace. Efficient for
  multi-tenant APIs that return the same data to all users in a tenant.
- `"client-id"` — each user has their own namespace. Use when the upstream returns per-user data.

`client-id`, `tenant-id`, and `claim:` are **inert** until `token-auth` runs on the same route
and populates `ctx.identity`. On a route with no auth plugin, every request will BYPASS.

---

## Alongside an external cache

The cache lives in the memory of each Flow instance. It is cleared on restart and not shared between instances. For a cache shared across instances or kept across restarts, run an external cache alongside Flow:

```
Flow (node 1) ─┐
               ├─> [Redis / CDN] ──> [Backends]
Flow (node 2) ─┘
```

Or use a CDN for static content next to Flow for dynamic API traffic:

```
Internet
  |
  +---> [CDN] (static assets, cached responses)
  |
  +---> [Flow] (dynamic API requests, auth, routing)
```

---

## Related

- [Token authentication](token-auth.md) — required upstream of `identity-from "client-id"`, `"tenant-id"`, and `"claim:"`; configures which claims map to `client_id` and `tenant_id`
- [Rate limiting](rate-limiting.md) — use alongside `cache-response` to protect cached routes from abuse; `key-from "client-id"` uses the same identity set by `token-auth`
- [Policies](../policies.md)
