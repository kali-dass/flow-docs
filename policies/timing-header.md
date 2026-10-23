# Timing header

Adds latency timing headers to the response, so you can see where a request's time went.

**Phase:** `upstream-response`

```kdl
path-control {
    upstream-response {
        filter kind="timing-header"
    }
}
```

It takes no options.

## What you get

| Header | Meaning |
|---|---|
| `X-Upstream-Duration-Us` | Time until your backend's response headers arrived — mostly how long your **backend** took. It also includes any `upstream-request` policies, and the time a slow client takes to upload a request body that the backend waits for |
| `X-Flow-Duration-Us` | Flow's own overhead — how long **Flow** took, not counting the time to connect to your backend. It includes time a policy spends waiting on an external service (for example token introspection) |

Both are in **microseconds** (µs), not milliseconds. Flow's own overhead is routinely well
under a millisecond, so milliseconds would round most of it away.

## Reading the numbers

The two headers split a request's time into the part you can fix and the part your backend
owns:

- **`X-Upstream-Duration-Us` is large** → your backend is slow. Flow is waiting on it.
- **`X-Flow-Duration-Us` is large** → the time is going into Flow's own work: routing,
  policies, TLS, or a policy waiting on an external service.

In a healthy setup `X-Flow-Duration-Us` is tens of microseconds and `X-Upstream-Duration-Us`
dominates.

The two headers do not add up to the whole request. Two things are in neither number: the time
to get a connection to your backend (reusing a pooled connection is fast, but opening a new
TCP or TLS connection can take milliseconds), and the time to send the response to the client.
If a request feels slower than the headers explain, a new backend connection on that request
is the usual cause: check that connections are being reused. If Flow's own number is large, look at how many policies the route runs and whether
TLS is enabled on both hops.

## Trying it out

```bash
curl -sD - -o /dev/null http://localhost:8080/anything | grep -i duration
```

```
x-upstream-duration-us: 1843
x-flow-duration-us: 62
```

Here the backend took 1.84 ms and Flow added 62 µs.

## Placement

For `X-Flow-Duration-Us` to capture *all* response-side processing, place `timing-header`
**last** in the `upstream-response` block — any filter after it runs after the clock is read
and won't be counted (the miss is only a few µs of cheap header work).

## Cost and when to use it

This filter only sets two response headers — it's cheap and safe to leave enabled. The reason
to gate it is **disclosure, not performance**: `X-Flow-Duration-Us` / `X-Upstream-Duration-Us`
expose your internal latency to whoever receives the response. Enable it where that's fine
(internal services, debugging) and leave it off on responses to untrusted clients.

For ongoing latency *monitoring*, prefer aggregate percentiles measured at your load balancer
or client rather than reading a header per request.

## Related

- [Performance](../performance.md) — what limits Flow's throughput, and how to measure it
- [Policies index](../policies.md)
