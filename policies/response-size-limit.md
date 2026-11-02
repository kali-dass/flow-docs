# Response size limit

Stops a response whose body is larger than a limit you set from reaching the client. It is a safeguard against bulk data extraction through a single call, for example a compromised API credential used to pull far more than a normal call returns. A response over the limit is refused or cut off; it is never delivered in full with a warning. Phase: `upstream-response`.

## Usage

```kdl
path-control {
    upstream-response {
        filter kind="response-size-limit" max-body-size="10MB"
    }
}
```

| Option | Required | Description |
|---|---|---|
| `max-body-size` | yes | The largest response body delivered. A whole number of bytes (`1048576`) or a number with a suffix: `KB`, `MB` or `GB`, case-insensitive. Suffixes are binary: `1KB` is 1,024 bytes and `1MB` is 1,048,576 bytes. Must be greater than 0. There is no "unlimited" value; to apply no limit, leave the policy off. |
| `override-service-limit` | no | `true` on a route-level declaration makes the route's `max-body-size` replace the service-level one instead of applying together with it. See [Service and route limits](#service-and-route-limits). Default `false`. |

A body of exactly `max-body-size` bytes is delivered; one byte more is not. The limit is per response, never across requests. Flow counts the bytes exactly as the backend sent them and never decompresses, so a compressed response counts at its compressed size. If your backend compresses, the client can expand those bytes into more data than the limit; set the limit with the compression ratio in mind, or turn backend compression off on the route.

## What happens when a response is too large

What the client sees depends on whether the backend declared the size up front.

| Backend response | When the check happens | What the client gets |
|---|---|---|
| Declares a `Content-Length` over the limit | Immediately, before any header is sent to the client | A clean `502` from Flow, and no body bytes |
| No declared length (chunked, close-delimited, or an HTTP/2 response without `content-length`), or a declared length within the limit | As the body arrives; the running total is checked on every piece | The response starts normally, then is cut off when the next piece would cross the limit |

A refusal before any header is sent is a `502` with a JSON body, `Server: Flow` and `Cache-Control: no-store`:

```json
{"error":"bad_gateway","message":"Upstream response exceeds the configured limit"}
```

**A response cut part-way.** Once the headers are sent the status cannot change, so the client keeps the status it was given (usually `200`) and sees a failed transfer. On HTTP/1.1 the chunked body ends without its terminating chunk and the connection is closed; on HTTP/2 the stream is reset. A well-behaved client reports this as an incomplete response. With `curl` 8.7.1 the HTTP/1.1 case ends with exit code 18 (`transfer closed with outstanding read data remaining`) and the HTTP/2 case with exit code 92 (`HTTP/2 stream 1 was not closed cleanly: CANCEL`), after the `200` and the bytes already delivered; a refusal is an ordinary `502`, so `curl` exits 0 and reports it as the status. Flow drops the piece that would cross the limit whole, so the client receives at most `max-body-size` bytes, sometimes slightly fewer. Flow also closes its connection to the backend, which stops sending.

Further behavior:

- **No body, no check.** `HEAD` responses, `204`, `304` and `1xx` carry no body, so a `Content-Length` on them is not judged.
- **Range requests.** A `206` response is judged by the length of the part being sent, not by the full resource. Someone fetching a large file in many small ranges stays under the limit on each call; use [rate limiting](rate-limiting.md) for repeated requests.
- **Malformed backend `Content-Length`.** A value that is not a plain whole number, or several `Content-Length` headers that disagree, is rejected before this policy runs and the client gets a `502`. Identical duplicates (`42, 42`) are accepted as one value.
- **A backend that overruns its own length.** On HTTP/1.1 only the declared number of bytes is read. On HTTP/2 the stream is reset and nothing is delivered.
- **Streams.** SSE and newline-delimited JSON responses are counted like any other, with no exemption by content type: a client could otherwise ask for a streaming format to take unlimited data. A route meant to stream for a long time should not carry this policy, or should carry a limit sized for one call; see [Service and route limits](#service-and-route-limits).
- **Cache.** A response over the limit is never stored by [response caching](cache-response.md), so a cache hit is always served from a response that passed the limit.

## Service and route limits

The policy can be set on a whole service, on a single route, or both.

- **Service-level.** Applies to every route in the service.
- **Route-level.** Applies to that route, in addition to the service-level one. When both apply, the lower limit blocks: a route can never silently raise the service limit.
- **Route-level with `override-service-limit=true`.** The route's `max-body-size` replaces the service-level one for that route, higher or lower. This is how a route that legitimately returns large bodies (an export, a download) gets a larger limit while the rest of the service stays strict. A long-lived stream route can do the same, with a number sized for it.

Flow refuses to start when `override-service-limit=true` is set on a service-level declaration, or on a route in a service that has no service-level declaration (there is nothing to override). Flow logs each overriding route at startup, so raised limits are easy to find in a review.

## What the limit does not cover

- **gRPC routes.** A gRPC call is a stream of messages, so a total-bytes cap would cut healthy long calls. Flow refuses to start if `response-size-limit` is set on a route declared `allowed-protocols "grpc"` or `"grpcs"` (on the route or inherited from the service). See [Non-REST Protocols](../non-rest-protocols.md#grpc).
- **WebSocket.** The upgrade response is not a body. Once the connection is upgraded, nothing sent over it is counted, so a WebSocket route is not limited by this policy.
- **Many calls.** The limit bounds one response. Extraction spread across many calls is for [rate limiting](rate-limiting.md).

## Example

A service that caps responses at 5 MB, with one export route allowed up to 200 MB:

```kdl
services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        path-control {
            upstream-response {
                filter kind="response-size-limit" max-body-size="5MB"
            }
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
            }
            route {
                name "export"
                match {
                    path-prefix "/export"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                path-control {
                    upstream-response {
                        filter kind="response-size-limit" max-body-size="200MB" override-service-limit=true
                    }
                }
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

## Observability

Every block is visible; a limit that silently looks like a normal `200` would hide the very behavior it exists to stop.

- **Metric.** `flow_size_limit_blocked_total{service, route, direction="response", action}` counts blocks (the same counter [Request size limit](request-size-limit.md) uses with `direction="request"`). `action` is `refused` (before any header was sent) or `truncated` (cut part-way). See [Metrics](../metrics.md#size-limits).
- **Request counts.** A refusal counts in `flow_requests_total` as `5xx` with `source="flow"`. A cut response counts with the status the client received, usually `2xx`, so use the metric above to see it.
- **Log.** Each block writes an error-level line with the client IP, the client identity when known, the route, the limit and where it came from (route or service), the declared or counted size, and the correlation ID.
