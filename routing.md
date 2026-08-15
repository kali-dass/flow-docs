# Routing

Flow picks exactly one route per request. This page covers the matchers you can write and
how Flow decides which route wins when several match.

## The `match` block

Every route has a `match` block containing one matcher.

```kdl
route {
    match {
        path-prefix "/api"
    }
    ...
}
```

## Matchers

### `path-prefix`

Matches when the URI path starts with the given string.

```kdl
match { path-prefix "/api/v2" }
```

Matches `/api/v2`, `/api/v2/users`, `/api/v2/anything`. Does **not** match `/api/v21`
only in the sense that a more specific route would win — as a raw prefix, `/api/v2` *does*
match `/api/v21`. Use `path-regex` if you need a boundary.

### `path-regex`

Matches when the URI path satisfies a regular expression.

```kdl
match { path-regex "^/static/.*\\.(js|css|png|svg)$" }
```

### `header`

Matches when a named request header's value satisfies a regex.

```kdl
match {
    header name="X-Tenant" value-regex="^acme-.*$"
}
```

Useful for multi-tenant routing and API versioning.

### `sni`

Exact match on the TLS SNI (or the `Host` header on plain HTTP), with the port stripped.

```kdl
match { sni "payments.example.com" }
```

### `sni-suffix`

Suffix match on the SNI / `Host`.

```kdl
match { sni-suffix ".internal.example.com" }
```

Matches `payments.internal.example.com`, `auth.internal.example.com`, and so on.

### `all` — logical AND

Every sub-matcher must match. Scores are **summed**.

```kdl
match {
    all {
        path-prefix "/secure/data"
        header name="Authorization" value-regex="^Bearer .+$"
    }
}
```

A request must have *both* the path prefix and a bearer token. Missing either means no
match.

### `any` — logical OR

The highest-scoring sub-matcher wins. Score is the **max** of the components.

```kdl
match {
    any {
        path-prefix "/health"
        path-prefix "/ping"
    }
}
```

---

## How ties are broken: scoring

When multiple routes match a request, Flow computes a **score** for each and picks the
highest. The rules:

| Matcher | Score |
|---|---|
| `sni` (exact host) | `usize::MAX` — **always wins** |
| `sni-suffix` | Length of the suffix |
| `path-prefix` | Length of the prefix — **longer prefix wins** |
| `path-regex` | Byte length of the matched region |
| `header` | Fixed weight of **500** |
| `all` | **Sum** of its components |
| `any` | **Max** of its components |

Three consequences worth internalizing:

1. **A longer path prefix always beats a shorter one.** `/api/v2` (7) beats `/api` (4),
   which beats `/` (1). This is what makes layered routes work naturally.

2. **A header match outweighs most path matches.** Its fixed 500 is far above any realistic
   path length, so a route matching on a header takes precedence over one matching only on
   a path. If you want a header-matched route to *lose* to a specific path, combine them
   with `all` so the scores add.

3. **An exact SNI match beats everything.** Use it when a hostname must be routed
   unconditionally.

## Methods and protocols are filters, not matchers

`allowed-methods` and `allowed-protocols` are **not** part of scoring. They are checked
after a route matches:

- A request whose **method** isn't in `allowed-methods` **does not match the route at all**
  — Flow keeps looking, and returns **502** if nothing else matches.
- A request whose **protocol** isn't in `allowed-protocols` is rejected with **426**
  (plain HTTP where TLS is required) or **400** (wrong HTTP version).

## No match

If no route matches, Flow returns **502**.

---

## Trying it out

These examples assume a service with a plain HTTP listener on `:8080` and a TLS listener on
`:4443`. Every example is shown for both. `-k` tells curl to accept a self-signed
certificate.

### Path prefix

```bash
# HTTP
curl -v http://localhost:8080/anything/api/v2/users

# HTTPS
curl -kv https://localhost:4443/anything/api/v2/users
```

With routes on both `/anything/api` and `/anything/api/v2`, the **longer prefix wins** — the
`/v2` route handles this request.

```bash
# Falls to the shorter /anything/api route
curl -v http://localhost:8080/anything/api
curl -kv https://localhost:4443/anything/api
```

### Header

```bash
# Matches a route with header name="X-Tenant" value-regex="^acme-.*$"
curl -v http://localhost:8080/anything -H "X-Tenant: acme-test"
curl -kv https://localhost:4443/anything -H "X-Tenant: acme-test"

# No match — different tenant, falls through (502 if nothing else matches)
curl -v http://localhost:8080/anything -H "X-Tenant: globe-test"
```

### Header beats path

```bash
curl -v http://localhost:8080/anything/api/v2/users -H "X-Tenant: acme-test"
curl -kv https://localhost:4443/anything/api/v2/users -H "X-Tenant: acme-test"
```

Even though the path matches a `/anything/api/v2` route, the **header route wins** — its
fixed score of 500 outweighs the 15-character path prefix.

### AND (`all`)

```bash
# Matches: has both the path and the bearer token
curl -v http://localhost:8080/anything/secure/data -H "Authorization: Bearer eyJhbGciOi..."
curl -kv https://localhost:4443/anything/secure/data -H "Authorization: Bearer eyJhbGciOi..."

# No match: right path, missing header
curl -v http://localhost:8080/anything/secure/data
```

### OR (`any`)

```bash
curl -v http://localhost:8080/anything/health
curl -v http://localhost:8080/anything/ping
curl -kv https://localhost:4443/anything/health
```

Both paths hit the same route.

### Path regex

```bash
curl -v http://localhost:8080/anything/static/app.js
curl -v http://localhost:8080/anything/static/app.css
curl -kv https://localhost:4443/anything/static/app.png
```

### SNI and `Host`

On **HTTPS**, `sni` matches the TLS SNI. On **plain HTTP** there is no SNI, so `sni` matches
the **`Host` header** instead — which is why these work over HTTP:

```bash
# HTTP — the Host header stands in for SNI
curl -v http://localhost:8000/ -H "Host: onevariable.com"

# HTTPS — the SNI must actually match; a spoofed Host header will not help
curl -kv https://localhost:8443/ -H "Host: onevariable.com"
```

That second command **fails to match**: curl sends SNI `localhost` (from the URL) while the
`Host` header claims `onevariable.com`. On TLS, Flow matches the SNI, not the header.

### SNI suffix

```bash
curl -v http://localhost:8000/get -H "Host: payments.internal.example.com"
curl -v http://localhost:8000/get -H "Host: auth.internal.example.com"
```

Both match a route with `sni-suffix ".internal.example.com"`.

### Methods

```bash
# Matches a route with allowed-methods "GET"
curl -v -X GET http://localhost:8080/anything/test

# Not in allowed-methods → the route does not match at all → 502
curl -v -X POST http://localhost:8080/anything/test

# Non-standard methods work if listed
curl -v -X PURGE http://localhost:8080/anything/test
```

Method names are case-insensitive.

### Protocols

This is the one that surprises people. Given a route with:

```kdl
allowed-protocols "https" "h2"
```

then:

```bash
# Works — TLS
curl -kv https://localhost:4443/anything/test

# 426 Upgrade Required — the route requires TLS, this is plain HTTP
curl -v http://localhost:8080/anything/test
```

A protocol mismatch is **not** a routing miss. The route matched; the request was then
rejected. You get **426** (TLS required) or **400** (wrong HTTP version) — not a 502. Add
`"http"` and `"h1"` to `allowed-protocols` if the route should accept plain HTTP.

---

## Performance note

Flow classifies your routes at startup and picks a caching strategy automatically:

| Your routes use | Strategy |
|---|---|
| Only path prefixes | A pre-built prefix index — fastest |
| Any regex | Two-level cache, optimized for repeated paths |
| Complex combinations | Full scan on every request |

You don't configure this, but it's worth knowing that **prefix-only routing is the fastest**
and heavy use of `all`/`any` combinators pushes a service into the full-scan path. If a
service is latency-critical and has many routes, prefer plain prefixes where you can.

The dynamic cache size is tunable per service with `route-cache-dynamic-capacity`.
