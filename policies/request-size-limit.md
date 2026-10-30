# Request size limit

Rejects requests whose body is larger than a limit you set, before the backend is asked to handle them. Phase: `request-filters`.

## Usage

```kdl
path-control {
    request-filters {
        filter kind="request-size-limit" max-body-size="10MB"
    }
}
```

| Option | Required | Description |
|---|---|---|
| `max-body-size` | yes | The largest request body accepted. A whole number of bytes (`1048576`) or a number with a suffix: `KB`, `MB` or `GB`, case-insensitive. Suffixes are binary: `1KB` is 1,024 bytes and `1MB` is 1,048,576 bytes. Must be greater than 0. |

A body of exactly `max-body-size` bytes is accepted; one byte more is rejected. The limit applies to the bytes on the wire, so a compressed body counts at its compressed size. Flow does not decompress request bodies.

## What happens when a request is too large

Flow answers `413` with a JSON body:

```json
{"error":"payload_too_large","message":"Request body exceeds the configured limit"}
```

| Request | When the check happens | Backend contacted? |
|---|---|---|
| Declares a `Content-Length` over the limit | Immediately, before any body is read | No |
| Declares an acceptable length, or none (chunked, or an HTTP/2 upload with no length) | As the body arrives; the total is checked on every piece | Yes, up to the point the limit is crossed. See below. |

- **`Expect: 100-continue`.** A client that asks permission before uploading gets `413` straight away, with no `100 Continue`, so it never sends the body. Flow never answers a size rejection with `417`: clients retry a `417` without the expectation, which would send the whole oversized body.
- **Malformed `Content-Length`.** A value that is not a plain whole number, or several `Content-Length` headers that disagree, is rejected with `400`. Most malformed values are refused by Flow's HTTP parser with a plain `400` before the policy runs.
- **Connection.** After a `413` on an HTTP/1.1 connection, Flow closes the connection (`Connection: close`) rather than reuse it, because the unread body is still arriving on it. On HTTP/2 only the one request is ended; the connection carries on.
- **A body cut part-way.** If a body with no declared length crosses the limit midway, the client gets `413`. The part sent before that point has already gone to the backend, so the backend sees a request whose body stops early and whose connection is then closed, the same as when a client disconnects mid-upload. A declared length over the limit is refused before the backend sees anything.

## What the limit does not cover

- **gRPC routes.** A gRPC call is a stream of messages, so a total-bytes cap would cut healthy long calls. Flow refuses to start if `request-size-limit` is set on a route declared `allowed-protocols "grpc"` or `"grpcs"` (on the route or inherited from the service). See [Non-REST Protocols](../non-rest-protocols.md#grpc).
- **WebSocket.** The upgrade request is limited as usual. Once the connection is upgraded, nothing sent over it is counted, so a long WebSocket session is never cut by this policy. See [Non-REST Protocols](../non-rest-protocols.md).
- **Responses.** The limit applies to request bodies only. Streaming responses (SSE, newline-delimited JSON) are unaffected; a request that starts one is limited like any other.
- **Long uploads.** A client that streams one very long request (for example continuous log shipping over a single request) is cut when the total passes the limit. Leave the policy off such a route, or set a limit large enough for it.

## Placement

Filters run in the order written, and service-level filters run before route-level ones. Put `request-size-limit` after your authentication policy (in the same block, or at route level when authentication is route-level), so callers who are not authenticated never learn the limit from a `413`. This costs no bandwidth: authentication works on headers, so an unauthenticated request is refused before any body is read.

## Example

A complete route that accepts uploads up to 5 MB:

```kdl
services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "uploads"
                match {
                    path-prefix "/uploads"
                }
                allowed-methods "POST" "PUT"
                allowed-protocols "http" "h1"
                path-control {
                    request-filters {
                        filter kind="request-size-limit" max-body-size="5MB"
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

A rejection counts in `flow_requests_total` with `status_class="4xx"` and `source="flow"`, so it is separate from errors the backend returned. See [Metrics](../metrics.md).
