# CORS

Lets web pages served from one site call your API on another. Browsers block a page from reading a response from a different **origin** (the scheme, host and port: `https://app.example.com` and `https://api.example.com` are different origins) unless the response says it may. For calls that could change data, the browser first sends a **preflight**, a harmless `OPTIONS` request asking whether the real call is allowed. `cors` makes Flow answer both, so your backends do not have to.

You need this when a web page on one origin calls an API on another and the browser console reports "blocked by CORS policy". You do not need it for calls from servers, mobile apps using native HTTP, or `curl`: only browsers enforce CORS.

CORS is a browser rule, not access control: `curl` and server-side clients ignore it, and it does not stop a cross-site `POST` that carries cookies. Keep authentication, [rate limiting](rate-limiting.md) and the [size limits](request-size-limit.md) in place.

## Quick start

Allow one web app to call one route. The route must list `OPTIONS`, because that is how the browser sends its preflight.

```kdl
services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET" "POST" "OPTIONS"
                allowed-protocols "http" "h1"
                cors {
                    allow-origins "https://app.example.com"
                }
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

Check it without a browser. A request that asks the question a browser would ask:

```bash
curl -i -X OPTIONS http://localhost:8080/orders \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST"
```

The answer is `204` with `Access-Control-Allow-Origin: https://app.example.com`. Repeat it with `-H "Origin: https://other.example"`: you still get `204`, but with no `Access-Control-Allow-*` headers, and a browser would refuse the call.

## Common setups

| You have | Configure |
|---|---|
| A single-page app on one origin calling your API | `allow-origins "https://app.example.com"` |
| The app sends an `Authorization` header or a JSON body | Add `allow-headers "Authorization" "Content-Type"`; headers outside the default list must be named |
| The app sends cookies or an `Authorization` header and needs to read the answer | `allow-credentials true` with the exact origins (never `"*"`) |
| Several apps on subdomains of one domain | `allow-origins "https://*.example.com"` |
| A public API that anyone's page may call, with no cookies | `allow-origins "*"` |
| Most routes need CORS, a few do not | Put `cors` on the service and `cors { enabled false }` on the exceptions |
| The page must read a custom response header | `expose-headers "X-Total-Count"` |

## Usage

`cors` is a block on a **service** (the default for every route) or on a **route** (it replaces the service's block for that route). Write it next to `routes`, not inside `path-control`.

```kdl
services {
    Api {
        listeners {
            "0.0.0.0:8443" cert-path="/etc/flow/cert.pem" key-path="/etc/flow/key.pem" offer-h2=true
        }
        cors {
            allow-origins "https://app.example.com" "https://*.partner.example.net"
            allow-headers "Authorization" "Content-Type"
            allow-credentials true
            max-age-secs 3600
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET" "POST" "DELETE" "OPTIONS"
                allowed-protocols "https" "h2"
                connectors {
                    "10.0.0.5:8080"
                }
            }
            route {
                name "public-widget"
                match {
                    path-prefix "/widget"
                }
                allowed-methods "GET" "OPTIONS"
                allowed-protocols "https" "h2"
                cors {
                    allow-origins "*"
                }
                connectors {
                    "10.0.0.6:8080"
                }
            }
            route {
                name "internal"
                match {
                    path-prefix "/internal"
                }
                allowed-methods "GET"
                allowed-protocols "https" "h2"
                cors {
                    enabled false
                }
                connectors {
                    "10.0.0.7:8080"
                }
            }
        }
    }
}
```

| Setting | Default | Description |
|---|---|---|
| `allow-origins` | required | The origins allowed to call the route: exact origins (`https://app.example.com`), subdomain wildcards (`https://*.example.net`), or `"*"` alone for every origin. There is no default, so an unset list can never mean "everyone". |
| `allow-methods` | the route's `allowed-methods` | The methods a browser may use. A request with another method gets no CORS headers. |
| `allow-headers` | `accept`, `content-type`, `x-requested-with` | The request headers a browser may send. `"*"` accepts any header the browser asks for. An explicit list replaces the default list and is matched case-insensitively. |
| `expose-headers` | none | Response headers that page code may read, in addition to the ones below. Browsers let a page read only a few standard headers otherwise. |
| `expose-flow-headers` | `true` | Also exposes `Retry-After` and, when the route's [correlation ID](correlation-id.md) is echoed to the client (`echo-downstream=true`), the correlation ID header. `false` exposes only `expose-headers`. |
| `allow-credentials` | `false` | Lets the page send cookies and `Authorization` headers. Not allowed with `allow-origins "*"`. |
| `max-age-secs` | none | How long a browser may remember a preflight answer. Browsers cap this at a few hours. |
| `preflight-status` | `204` | `204` or `200`, for clients that mishandle `204`. |
| `allow-private-network` | `false` | Answers a browser's private-network request (a public page calling an address on a private network) with `Access-Control-Allow-Private-Network: true`. |
| `enabled` | n/a | Route level only. `cors { enabled false }` opts the route out of the service's block. It cannot be combined with other settings. |

## Origins

An exact origin is `scheme://host[:port]`: no path, query or trailing slash. The scheme and host are case-insensitive, and the scheme's default port (80 for `http`, 443 for `https`) may be written or left out. The literal `null` (sent by sandboxed frames and `file:` pages) matches only if you list it. Any other scheme is accepted, for example `capacitor://localhost` for a mobile app.

A wildcard origin is `scheme://*.domain[:port]`. The `*` stands for one or more whole labels in front of the domain, so `https://*.example.net` allows `https://app.example.net` and `https://a.b.example.net`, and does not allow `https://example.net`, `https://evilexample.net`, `https://app.example.net.evil.test`, another scheme, or another port. Flow refuses to load a wildcard written any other way (`https://*example.net`, two wildcards, a wildcard on an IP address or a bare top-level domain such as `https://*.com`). There is no pattern or regular-expression matching.

When the answer depends on the origin, Flow adds `Vary: Origin` so that caches keep one answer per origin. With `allow-origins "*"` the answer is the same for everyone, so Flow sends `Access-Control-Allow-Origin: *` on every response, whether or not the request had an `Origin`, and no `Vary`.

## Preflight requests

Before a request that is not simple (a `PUT`, `DELETE`, `PATCH`, or a request with an `Authorization` or JSON `Content-Type` header), the browser sends a preflight: an `OPTIONS` request with `Origin` and `Access-Control-Request-Method` headers. Flow answers it itself and does not forward it to the backend.

| Situation | Response |
|---|---|
| The origin, the requested method and every requested header are allowed | `204` (or `preflight-status`) with `Access-Control-Allow-Origin`, `-Allow-Methods` (the configured methods), `-Allow-Headers` (the headers asked for), `-Allow-Credentials` and `-Max-Age` as configured |
| The origin, the method or a header is not allowed | The same status with no `Access-Control-Allow-*` headers: the browser blocks the call, and nothing about your configuration is revealed |
| An `OPTIONS` request without `Origin` or without `Access-Control-Request-Method` | An ordinary request: it goes through the route's policies to the backend |

A preflight is answered before the request filters, so it needs no token and is not counted by a rate limit or refused by an IP restriction; it carries no credentials and returns only configuration. The real request that follows is subject to every policy.

The route must accept `OPTIONS`: list `"OPTIONS"` in its `allowed-methods`. Flow never adds it for you.

## Responses from the backend

For a request with an allowed origin Flow adds `Access-Control-Allow-Origin` (the matched origin), `Access-Control-Allow-Credentials` when enabled, and `Access-Control-Expose-Headers`. A request from another origin, or with no `Origin`, gets no `Access-Control-Allow-Origin` and the request itself is still served: the browser, not Flow, decides to hide the response.

On a route with `cors`, Flow owns every `Access-Control-*` response header. Those the backend sent are removed and replaced by Flow's, because a response with two `Access-Control-Allow-Origin` values is rejected by browsers. Other backend headers are untouched, and a backend `Vary` is kept with `Origin` appended. On a route without `cors`, the backend's headers pass through unchanged.

## Responses Flow generates

Errors Flow produces itself carry the same headers for an allowed origin, so a page can read the real status instead of a CORS failure: `401` and `403` from [token authentication](token-auth.md) and [IP restriction](ip-restriction.md), `429` from [rate limiting](rate-limiting.md) (with `Retry-After` readable), `413` from the [request size limit](request-size-limit.md), [request termination](request-termination.md), and the `502` when the backend cannot be reached or a response is refused by the [response size limit](response-size-limit.md).

A `405` (method not allowed), the `502` for a path that matches no route, and `501` have no matched route to take a policy from. They carry the headers only when the **service** has a `cors` block. A page cannot read non-standard headers on these responses, such as `WWW-Authenticate`, unless you list them in `expose-headers`.

## Response caching

On a route that also has [`cache-response`](cache-response.md), responses that vary by origin are cached per origin, and a cache hit carries the `Access-Control-*` headers for the origin that asked. The cached copy has Flow's headers, not the backend's.

## Troubleshooting

When a browser says a call "has been blocked by CORS policy", its message names the cause. The common ones:

| The browser says | Cause | Fix |
|---|---|---|
| No `Access-Control-Allow-Origin` header is present | The page's origin is not in `allow-origins`. The match is exact: `http` and `https`, `www.` and without, and different ports are different origins | Add the exact origin the browser shows, or a wildcard for subdomains |
| Request header `authorization` (or another) is not allowed by `Access-Control-Allow-Headers` | The preflight asked for a header outside the list | Add it to `allow-headers`. The default list is only `accept`, `content-type` and `x-requested-with` |
| Method `PUT` (or another) is not allowed | The method is not in `allow-methods` | Add it (and to the route's `allowed-methods`) |
| The value of `Access-Control-Allow-Origin` must not be the wildcard `*` when the request's credentials mode is `include` | The page sends cookies and the route allows `*` | List the origins and set `allow-credentials true` |
| The call works but the page cannot read a response header | Browsers hide non-standard headers | Add the header to `expose-headers` |
| Works with `curl`, fails in the browser | `curl` does not enforce CORS | Test with the preflight command above and compare the headers |
| A `401` or `429` appears as a CORS error | The origin was not allowed, so the browser hides every response, errors included | Fix the origin; for an allowed origin Flow adds the headers to its own errors |

## Refused when the configuration loads

Flow refuses to start, naming the route, when:

- `allow-origins` is missing, mixes `"*"` with other origins, or has an origin with a path, a query, a trailing slash or a wrong scheme, or a badly placed wildcard.
- `allow-credentials true` is combined with `allow-origins "*"`.
- A route where `cors` applies, written on the route or inherited from the service, does not list `"OPTIONS"` in `allowed-methods`. A route that should not do CORS opts out with `cors { enabled false }`.
- A route declared `grpc` or `grpcs` has `cors`, written or inherited. Browsers do not make native gRPC calls; add `cors { enabled false }` to such routes under a service-level block.
- `cors` is written inside `path-control`, has an unknown or repeated setting, or `enabled` is used at service level or with other settings.

Flow starts with a warning when `allow-credentials true` is combined with a wildcard origin: any subdomain, including one that was taken over or hosts user content, could then read credentialed responses. List the exact origins that need credentials instead. The warning also appears with `--validate-configs`.

## Observability

A denied preflight is logged at debug level with the reason (origin, method or header). Counters for preflights allowed and denied are not provided yet.
