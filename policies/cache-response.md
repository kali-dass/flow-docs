# Cache response

Caches upstream responses in memory and serves subsequent matching requests directly from the
cache, bypassing the backend entirely.

**Configuration:** `cache-response` is a route-level block — not a `path-control` filter.

```kdl
routes {
    route {
        match { path-prefix "/api/v1" }
        upstream "backend"
        cache-response {
            ttl-secs 60
            max-body-bytes 1048576
            max-capacity-bytes 104857600
            cache-status-codes 200 201        // multiple status codes: space-separated arguments
            allowed-methods "GET" "HEAD"      // multiple methods: space-separated string arguments
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
| `cache-status-codes` | `200` | One or more HTTP status codes eligible for caching, written as separate integer arguments: `cache-status-codes 200 201 204` |
| `allowed-methods` | `"GET" "HEAD"` | One or more HTTP methods eligible for cache lookup and storage, written as separate string arguments: `allowed-methods "GET" "HEAD"` |
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

An `Age` header is injected on `HIT`, indicating how many seconds ago Flow received this
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

## Headers stripped before storage

Hop-by-hop headers are stripped and never replayed: `Transfer-Encoding`, `Connection`,
`Keep-Alive`, `Upgrade`, `Via`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`,
and `Age`.

Rate-limit headers are also stripped — any header whose name begins with `RateLimit-`,
`X-RateLimit-`, or `X-Rate-Limit-` reflects a per-connection, per-time-window value and would
be misleading if replayed from a cached response.

## Identity isolation

By default all requests share the same cache key space. Use `identity-from` to scope the cache
key to a per-identity namespace — required when different authenticated users receive different
upstream responses.

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

## Related

- [Token authentication](token-auth.md) — required upstream of `identity-from "client-id"`, `"tenant-id"`, and `"claim:"`; configures which claims map to `client_id` and `tenant_id`
- [Rate limiting](rate-limiting.md) — use alongside `cache-response` to protect cached routes from abuse; `key-from "client-id"` uses the same identity set by `token-auth`
- [Policies](../policies.md)
