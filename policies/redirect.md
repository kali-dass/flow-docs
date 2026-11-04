# Redirect

Answers a request with an HTTP redirect and does not contact the backend. The client is told to ask somewhere else, and does. Use it to send callers from `http` to `https`, to move an API or a site to a new address, or to retire a path in favor of a new one. Phase: `request-filters`.

## Choosing a status

`status` is required and has no default, because the choice is real: whether the move is permanent, and whether the client keeps the request's method and body. Flow refuses to start without it and lists these choices in the error.

| Status | Meaning | Permanent? | What a client does with the request | Use it for |
|---|---|---|---|---|
| `301` | Moved Permanently | yes | May change a `POST` into a `GET` and drop the body | A page or site that moved for good, for browsers following links |
| `308` | Permanent Redirect | yes | **Keeps** the method and body | A permanent move of an API, where `POST`, `PUT` and `DELETE` must arrive unchanged |
| `302` | Found | no | May change a `POST` into a `GET` and drop the body | A temporary move for browsers following links |
| `307` | Temporary Redirect | no | **Keeps** the method and body | A temporary move of an API: a migration in progress, a test of the new address |
| `303` | See Other | no | Always follows with a `GET` | Sending a client to a result page after a `POST`; rarely wanted on a gateway |

In a test with `curl`, a `POST` followed `301`, `302` and `303` as a `GET` with no body, and followed `307` and `308` as a `POST` with its body. Clients differ in the details (`curl` keeps a `PUT` and its body on `301` and `302`, for example), so for an API choose `307` or `308`.

**Permanent redirects are remembered.** A browser may keep a `301` or `308` and go straight to the new address next time without asking Flow, so a wrong permanent redirect outlives the fix for everyone who saw it. Start with `307` and switch to `308` once the target is right. See [Caching of redirects](#caching-of-redirects).

## Usage

```kdl
path-control {
    request-filters {
        filter kind="redirect" status=308 location="https://api.example.com/v2" keep-path=true
    }
}
```

| Option | Default | Description |
|---|---|---|
| `status` | required | `301`, `302`, `303`, `307` or `308`. See [Choosing a status](#choosing-a-status). |
| `location` | required unless `upgrade-to-https` | Where to send the client: a full `http://` or `https://` URL, or a path starting with a single `/` (the client's own host). No fragment (`#...`), no user name, no spaces or backslashes. With `keep-query`, it must not contain a query of its own. |
| `keep-path` | `false` | Append the request's path to the path in `location`. |
| `keep-query` | `false` | Append the request's query string. Independent of `keep-path`. |
| `strip-prefix` | none | A path prefix removed from the request's path before it is appended, for example `"/api/v1"`. Needs `keep-path`. |
| `upgrade-to-https` | `false` | Send the request to `https` on the same host, keeping its path and query. Cannot be combined with `location`, `keep-path`, `keep-query` or `strip-prefix`. |
| `https-port` | `443` | The port written into the target when it is not `443`. Only with `upgrade-to-https`. |
| `cache-control` | `max-age=3600` for `301` and `308`; none for the others | The `Cache-Control` header of the redirect, written exactly as given. See [Caching of redirects](#caching-of-redirects). |

The response has the status, a `Location` header and `Content-Length: 0`, and no body.

## How the target is built

With neither `keep-path` nor `keep-query`, the client is sent to `location` exactly as written. Otherwise the target is built from `location` and the request:

| Request | Settings | Redirects to |
|---|---|---|
| `https://example.com/demo?foo=bar` | `location="https://new.example.com"`, `keep-path=true`, `keep-query=true` | `https://new.example.com/demo?foo=bar` |
| `https://example.com/demo?foo=bar` | `location="https://new.example.com/some-path"`, `keep-path=true`, `keep-query=true` | `https://new.example.com/some-path/demo?foo=bar` |
| `https://example.com/orders/1` | `location="https://api.example.com/v2"`, `keep-path=true` | `https://api.example.com/v2/orders/1` |
| `https://example.com/api/v1/orders/1` | `location="https://new.example.com"`, `keep-path=true`, `strip-prefix="/api/v1"` | `https://new.example.com/orders/1` |
| `https://example.com/old/page?lang=en` | `location="https://new.example.com/landing"`, `keep-query=true` | `https://new.example.com/landing?lang=en` |

Details:

- **Joining.** The request's path is added after the path in `location`, with exactly one `/` between them. A trailing `/` on `location` does not double up.
- **`strip-prefix`** removes the prefix only at a `/` boundary: `/api/v1` removes the start of `/api/v1/orders` but not of `/api/v10/orders`. A request that does not start with the prefix is appended unchanged.
- **The path and query are used exactly as the client sent them**, with no decoding, re-encoding or normalizing, so `%2F` stays `%2F`. An empty query (`/demo?`) is dropped; a trailing slash on the path is kept.
- **A request cannot choose where it is sent.** The host in the target is always the one in `location` (or, for `upgrade-to-https`, the request's own host). Leading slashes in the kept path collapse into one, so with `location="/"` a request for `//other.example/x` is sent to `/other.example/x` on your host, never to another site.
- **A request whose path or query contains a backslash, a space, a control character or a non-ASCII byte** is answered `400` and not redirected, because those characters could change where a browser goes.

Examples:

```kdl
// A retired page: one fixed place.
filter kind="redirect" status=301 location="https://www.example.com/pricing"

// A moved API: keep the path and the query.
filter kind="redirect" status=308 location="https://api.example.com" keep-path=true keep-query=true

// A moved version prefix: /api/v1/... becomes /...
filter kind="redirect" status=308 location="https://new.example.com" keep-path=true keep-query=true strip-prefix="/api/v1"
```

## Redirecting http to https

A route cannot tell a plain request from a TLS one for the same path, and a route that accepts only `https` answers a plain request with `426` before any filter runs. The way to redirect is a **separate service whose listener is the plain port**, next to the secure service:

```kdl
services {
    HttpRedirect {
        listeners {
            "0.0.0.0:80"
        }
        routes {
            route {
                name "to-https"
                match {
                    path-prefix "/"
                }
                allowed-methods "GET" "HEAD" "POST" "PUT" "PATCH" "DELETE" "OPTIONS"
                allowed-protocols "http" "h1"
                path-control {
                    request-filters {
                        filter kind="redirect" status=308 upgrade-to-https=true
                    }
                }
                connectors {
                    "127.0.0.1:1" disabled=true
                }
            }
        }
    }
    Secure {
        listeners {
            "0.0.0.0:443" cert-path="/etc/flow/cert.pem" key-path="/etc/flow/key.pem" offer-h2=true
        }
        routes {
            route {
                name "shop"
                match {
                    path-prefix "/"
                }
                allowed-methods "GET" "POST"
                allowed-protocols "https" "h2"
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

`upgrade-to-https` builds `https://<the request's host><path>?<query>`. The port the client used is dropped; add `https-port=8443` when the secure listener is on another port. The host is taken from the `Host` header (HTTP/1.1) or the authority (HTTP/2) and must be a plain host name or IP address: anything else is answered `400`.

Flow refuses to start if `upgrade-to-https` is on a route that `https` clients reach (`allowed-protocols` with `https`, or `h2` in a service that has a TLS listener), because they would be redirected to https again, forever.

## Caching of redirects

Browsers and proxies may keep a redirect and reuse it without asking Flow again. A mistaken redirect can then keep sending people to the wrong place after you fix the configuration.

- **`301` and `308`** get `Cache-Control: max-age=3600` (one hour) unless you set `cache-control`. That limits how long a mistake outlives its fix, and costs a permanent redirect one request per address per hour per browser.
- **`302`, `303` and `307`** get no header by default; browsers do not keep them without one.
- **`cache-control`** replaces the default on any status: `cache-control="no-store"` to never keep it, `cache-control="max-age=300"` for five minutes, `cache-control="public, max-age=31536000, immutable"` for a redirect you are sure of. It must be a valid header value.

To test a redirect, use `307` first, check the target, then switch to `308`. If a browser kept a wrong redirect while you were testing, clear its cache or use a private window.

## Where to put it in the filter list

`redirect` is a request filter, so filters run in the order written and the first one that answers ends the request.

- **First**, for a move that applies to everyone (http to https, a domain move).
- **After [token authentication](token-auth.md)**, for a retired address that only known callers should learn about: a caller without a valid token gets the `401`, not the new location.
- A filter written **after** a `redirect` never runs, because the redirect answers every request first. Flow starts, and logs a warning that names the filters (also shown by `--validate-configs`). The same is true after a [request termination](request-termination.md) without a `trigger`.
- A [CORS](cors.md) preflight is answered before the filters, so a redirect never answers a preflight. A redirect from a route with `cors` carries the CORS headers.
- A redirect is not valid on a gRPC route; Flow refuses to start.

## A route that only redirects

Every route needs a `connectors` block. A route that answers every request itself, like the one above, marks its connector `disabled=true`: Flow never resolves, health checks or uses it. See [Disabled connectors](../configuration.md#disabled-connectors). A request that reached a route with no usable connector would get a `502`.

## Troubleshooting

| You see | Likely cause | Fix |
|---|---|---|
| "Too many redirects" in the browser, or `curl` stops with an error after 50 | The target leads back to a redirect: `location` points into the same route, two redirects point at each other, or a redirect to `https` is reached over `https` | See the chain with `curl -IL --max-redirs 5 <url>` and follow each `Location` |
| The old redirect still happens after you removed it | The browser kept a `301` or `308` | Clear the browser's cache or use a private window; test with `307` |
| A `POST` arrives as a `GET` with no body | The status was `301`, `302` or `303` | Use `307` or `308` |
| Flow refuses to start and lists the five statuses | `status` is missing or is not one of them | Pick one from [Choosing a status](#choosing-a-status) |
| `400` instead of a redirect | The request's path, query or host holds a character that cannot be put in a `Location` header | Expected for malformed requests; check the client |

Flow checks the loops it can see when the configuration loads: a path-only `location` that the same route matches is a warning, and `upgrade-to-https` on a route that `https` clients reach is refused. A loop through other routes, services or systems cannot be seen from one configuration.
