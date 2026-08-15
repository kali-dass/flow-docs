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
| `X-Upstream-Duration-Us` | Time spent on the upstream round-trip — how long your **backend** took |
| `X-Flow-Duration-Us` | Flow's own overhead — how long **Flow** took |

Both are in **microseconds** (µs), not milliseconds. Flow's own overhead is routinely well
under a millisecond, so milliseconds would round most of it away.

## Reading the numbers

The two headers split a request's time into the part you can fix and the part your backend
owns:

- **`X-Upstream-Duration-Us` is large** → your backend is slow. Flow is waiting on it.
- **`X-Flow-Duration-Us` is large** → the time is going into Flow's own work: routing,
  policies, TLS.

In a healthy setup `X-Flow-Duration-Us` is tens of microseconds and `X-Upstream-Duration-Us`
dominates. If Flow's own number is large, look at how many policies the route runs and whether
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
