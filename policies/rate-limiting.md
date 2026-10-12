# Rate limiting

Caps how fast requests are accepted. A single `rate-limit` filter handles all rate-limiting
scenarios — `key-from` selects what gets its own budget:

| `key-from` | One budget per… | Use it to… |
|---|---|---|
| `"source-ip"` | client IP address | Stop one client overwhelming you |
| `"uri-group"` | everything matching a pattern | Cap total traffic to an endpoint group |
| `"uri"` | individual URI path | Stop one hot path starving the others |
| `"client-id"` | authenticated client | Per-user/per-tenant limiting (requires auth filter) |
| `"header:<name>"` | value of a named header | Limit by tenant ID header or similar |
| `"composite"` | combination of fields | Per-client-per-route or other multi-dimensional limits |

**Phase:** `request-filters`

## How the limiter works

All variants use a **token bucket**:

- The bucket holds up to `max-tokens`.
- Every request takes one token.
- The bucket refills by `refill-qty` every `refill-interval-ms`.
- When the bucket is empty, requests are **rejected immediately** with `429 Too Many Requests`
  and a `Retry-After` header.

### The three parameters and what they control

| Parameter | What it controls |
|---|---|
| `max-tokens` | **Bucket capacity** — the maximum number of tokens the bucket can hold at any time. Sets the burst size. Not tied to any duration. |
| `refill-qty` | Tokens added per refill interval. |
| `refill-interval-ms` | How often tokens are added, in milliseconds. |

Two rates fall out of those parameters:

- **Burst** = `max-tokens` — how many requests can be served back-to-back from a full bucket,
  with no gaps.
- **Sustained rate** = `refill-qty` ÷ `refill-interval-ms` × 1000 req/s

So `max-tokens=50`, `refill-qty=5`, `refill-interval-ms=200` allows a burst of 50, then
sustains **25 req/s**.

### `max-tokens` is a capacity, not a rate

`max-tokens` does not represent "tokens per second" or tokens per any time window. It is the
size of the bucket at a moment in time. Two consequences:

**Idle clients accumulate tokens up to `max-tokens` and stop.** A client that sends no requests
for a long time does not build up an unlimited balance — the bucket fills to `max-tokens` and
stays there. When that client sends a burst, it can consume at most `max-tokens` tokens before
throttling begins.

**Setting `refill-qty` above `max-tokens` wastes the excess.** Tokens added by a refill that
would push the bucket above `max-tokens` are silently discarded. For example,
`max-tokens=2 refill-qty=5 refill-interval-ms=2000` refills 5 tokens every 2 seconds, but the
bucket can never hold more than 2 — the other 3 are lost every cycle. The effective sustained
throughput is still 2 requests per 2-second window, not 5. To get 5 per window, set
`max-tokens` to at least 5.

### Arrival rate vs. sustained rate

Whether `429`s appear depends on whether the request arrival rate exceeds the sustained rate:

- **Arrival rate < sustained rate** — the refill keeps ahead of consumption; the bucket stays
  non-empty and requests pass through. Occasional `429`s can still appear in short bursts if
  requests arrive faster than the refill interval even when the average rate is fine.
- **Arrival rate > sustained rate** — the bucket drains. Once empty, only `refill-qty` requests
  pass through per `refill-interval-ms`, and the rest are rejected until the next refill.
- **Variable arrival rate** — a client that is sometimes fast and sometimes slow will see `200`s
  when it is slow (bucket recovers) and `429`s when it accelerates past the sustained rate.

**Example:** `max-tokens=5 refill-qty=3 refill-interval-ms=500` sustains 6 req/s. A client
sending one request every 200 ms (5 req/s) should stay under the limit. But if that client
responds to each reply immediately — no fixed sleep, just as fast as the network allows — the
effective rate can spike above 6 req/s during fast network windows, causing intermittent `429`s
even though the average rate looks fine. Adding a fixed delay (`sleep 0.2`) restores the
controlled 5 req/s and eliminates the `429`s.

### Sizing your limits

Set `max-tokens` to the burst you are willing to absorb — how many back-to-back requests a
client should be allowed before throttling begins. Set the refill pair to the sustained rate
you want to allow. They are independent:

- Large burst, slow refill → generous initial headroom, tight long-run cap
- Small burst, fast refill → immediate throttling of spikes, smooth sustained throughput

Make sure `max-tokens` ≥ `refill-qty`, otherwise the refill is partially wasted every cycle.

---

## `key-from="source-ip"` — per client IP

**One bucket per source IP.** Each client gets its own budget, so one noisy client cannot
starve the others.

```kdl
request-filters {
    filter kind="rate-limit" key-from="source-ip" \
        max-tokens=100 refill-qty=10 refill-interval-ms=100 \
        max-buckets=50000 threads=8
}
```

Each IP gets a burst of 100 and **100 requests/second** sustained.

> **Check what "source IP" means in your deployment.**
>
> By default Flow uses the source IP of the TCP connection. If Flow sits behind a proxy, CDN,
> or cloud load balancer, that is the *proxy's* IP — every client shares one bucket and the
> policy does the opposite of what you intended.
>
> To use the real client IP, configure `trusted-proxies` and `real-ip-source` at the system
> level. Flow then reads the IP from `X-Forwarded-For` (or a CDN header like
> `CF-Connecting-IP`) and each real client gets its own bucket:
>
> ```kdl
> system {
>     trusted-proxies {
>         cidr "10.0.0.0/8"              // your LB / proxy CIDR range
>     }
>     real-ip-source "xff"               // or "header:CF-Connecting-IP" for Cloudflare
> }
> ```
>
> `real-ip-source` requires `trusted-proxies` — Flow refuses to start without it. See
> [Proxy deployments and source IP](#proxy-deployments-and-source-ip) below.

### Proxy deployments and source IP

When Flow sits behind a proxy, CDN, or cloud load balancer, configure `trusted-proxies` and
`real-ip-source` so Flow resolves the real client IP from the forwarded headers:

```kdl
system {
    trusted-proxies {
        cidr "10.0.0.0/8"                  // your LB / proxy CIDR range
    }
    real-ip-source "xff"                   // read X-Forwarded-For
}
```

For CDN deployments, use the CDN's proprietary header instead of `xff`:

| CDN | `real-ip-source` value |
|---|---|
| Cloudflare | `"header:CF-Connecting-IP"` |
| AWS CloudFront | `"header:CloudFront-Viewer-Address"` |
| Fastly | `"header:Fastly-Client-IP"` |
| Akamai | `"header:True-Client-IP"` |
| Generic Nginx proxy | `"header:X-Real-IP"` |

**Security**: `real-ip-source` is a security-sensitive setting. Flow requires `trusted-proxies`
to be configured when `real-ip-source` is set to anything other than `"peer"` — it will refuse
to start without it. Only connections arriving from a declared trusted CIDR will have their
forwarded header trusted; direct connections always use the raw TCP peer, preventing header
injection.

---

## `key-from="uri-group"` — one shared bucket

**One shared bucket** for every request whose path matches `pattern`. All callers draw from the
same budget.

```kdl
request-filters {
    filter kind="rate-limit" key-from="uri-group" pattern="^/echo" \
        max-tokens=50 refill-qty=5 refill-interval-ms=200
}
```

Burst of 50, then **25 requests/second** shared across all clients.

Use it to protect a fragile backend from *total* load, regardless of who is calling.

---

## `key-from="uri"` — per URI path

**One bucket per unique URI path** matching `pattern`.

```kdl
request-filters {
    filter kind="rate-limit" key-from="uri" pattern="^/echo" \
        max-tokens=20 refill-qty=2 refill-interval-ms=500 \
        max-buckets=10000 threads=8
}
```

Each distinct path under `/echo` gets a burst of 20 and **4 requests/second** sustained.

---

## `key-from="client-id"` — per authenticated client

**One bucket per authenticated client.** Requires `token-auth` or `api-key-auth` to run
earlier in the same filter chain — the rate limiter reads the verified client identity set by
the auth filter.

```kdl
request-filters {
    filter kind="token-auth" { ... }
    filter kind="rate-limit" key-from="client-id" \
        max-tokens=200 refill-qty=20 refill-interval-ms=1000 \
        max-buckets=100000 threads=8
}
```

If a request reaches this filter without an authenticated identity, it is rejected with `401` —
not `429`. See [Missing key behavior](#missing-key-behavior).

---

## `key-from="header:<name>"` — per header value

**One bucket per value of a named request header.** Useful for tenant ID headers or similar
external identifiers.

```kdl
request-filters {
    filter kind="rate-limit" key-from="header:X-Tenant-ID" \
        max-tokens=500 refill-qty=50 refill-interval-ms=1000 \
        max-buckets=50000 threads=8
}
```

By default, a request without that header is rejected with `400`. Use `missing-key="skip"` to
pass it through instead, or `missing-key-status` to change the rejection code. See
[Missing key behavior](#missing-key-behavior) for full details and examples.

---

## `key-from="composite"` — multi-dimensional limit

**One bucket per combination of fields.** Use it to enforce separate limits per-client
per-route — for example, 1000 rps on `/api/search` but only 10 rps on `/api/export` for the
same client.

```kdl
request-filters {
    filter kind="rate-limit" key-from="composite" \
        max-tokens=10 refill-qty=1 refill-interval-ms=100 \
        max-buckets=50000 threads=8 {
        composite-fields {
            field name="client-id"
            field name="route-path"
        }
    }
}
```

By default, a request where any field is absent is rejected with `400`. Use `missing-key` and
`missing-key-status` to control this. See [Missing key behavior](#missing-key-behavior) for
full details and examples.

Available field names:

| Field | Bucket key contribution |
|---|---|
| `"source-ip"` | client IP address |
| `"uri"` | full URI path |
| `"client-id"` | authenticated client ID (requires auth filter) |
| `"route-path"` | matched route's path prefix |
| `"header:<name>"` | value of a named request header |

---

## Missing key behavior

Every `key-from` variant has a defined response when the key cannot be derived from the request:

| `key-from` | When the key is absent | Response | Configurable? |
|---|---|---|---|
| `uri-group` | Path does not match `pattern` | Pass through (filter doesn't apply to this path) | No |
| `uri` | Path does not match `pattern` | Pass through (filter doesn't apply to this path) | No |
| `source-ip` | Unix domain socket connection (no IP) | Pass through | No |
| `client-id` | No authenticated identity on the request | **Reject `401 Unauthorized`** | No |
| `header:<name>` | Named header is absent | **Reject `400 Bad Request`** (default) | Yes — `missing-key`, `missing-key-status` |
| `composite` | Any component field is absent | **Reject `400 Bad Request`** (default) | Yes — `missing-key`, `missing-key-status` |

For `uri-group` and `uri`, "absent key" means the path didn't match `pattern` — the filter was
never meant to govern that path, so pass-through is correct, not a failure.

For `client-id`, an unauthenticated request reaching a rate-limit filter is always a
configuration error (missing auth filter upstream) or an attack — `401` is hardcoded and cannot
be overridden.

For `header:<name>` and `composite`, the behavior is configurable:

- **`missing-key="reject"`** (default) — reject the request. Use `missing-key-status` to set
  the HTTP status code (any 4xx, 400–499; default `400`).
- **`missing-key="skip"`** — pass the request through without rate limiting. Use this when
  the header is optional and its absence is expected (for example, unauthenticated requests
  that a later filter will handle).

`missing-key-status` is only valid alongside `missing-key="reject"` — setting it with
`missing-key="skip"` is a configuration error.

**Example — header used as a required tenant identifier (`400` default):**

```kdl
filter kind="rate-limit" key-from="header:X-Tenant-ID" \
    max-tokens=500 refill-qty=50 refill-interval-ms=1000 \
    max-buckets=50000 threads=8
// Header absent → 400 Bad Request
```

**Example — header used as an auth token (`401` override):**

```kdl
filter kind="rate-limit" key-from="header:Authorization" \
    missing-key-status=401 \
    max-tokens=100 refill-qty=10 refill-interval-ms=1000 \
    max-buckets=50000 threads=8
// Header absent → 401 Unauthorized
```

**Example — optional header, skip when absent:**

```kdl
filter kind="rate-limit" key-from="header:X-Tenant-ID" \
    missing-key="skip" \
    max-tokens=500 refill-qty=50 refill-interval-ms=1000 \
    max-buckets=50000 threads=8
// Header absent → pass through, no rate limiting applied
```

**Example — composite with `client-id`, missing identity returns `401`:**

```kdl
filter kind="rate-limit" key-from="composite" \
    missing-key-status=401 \
    max-tokens=10 refill-qty=1 refill-interval-ms=100 \
    max-buckets=50000 threads=8 {
    composite-fields {
        field name="client-id"
        field name="route-path"
    }
}
// Any field absent (e.g. no authenticated client) → 401 Unauthorized
```

---

## Options

| Option | Applies to | Meaning |
|---|---|---|
| `key-from` | all | Dimension that identifies a bucket — see variants above |
| `max-tokens` | all | Bucket size and initial fill — the burst allowance |
| `refill-qty` | all | Tokens added per refill interval |
| `refill-interval-ms` | all | Refill period in milliseconds |
| `pattern` | `uri-group`, `uri` | Regex — filter only applies to paths that match |
| `max-buckets` | all multi-key variants | Maximum number of live buckets in memory |
| `threads` | all multi-key variants | Bucket cache shard count — set to match `threads-per-service` |
| `missing-key` | `header:<name>`, `composite` | What to do when the key field is absent: `"reject"` (default) or `"skip"` |
| `missing-key-status` | `header:<name>`, `composite` | HTTP 4xx status code (400–499) returned when rejecting a missing key; default `400`. Only valid with `missing-key="reject"`. |

`uri-group` uses one shared bucket — `max-buckets` and `threads` are not needed.
`client-id` hardcodes `401` for a missing identity — `missing-key` and `missing-key-status` do not apply.

### Multiple filters and stacking

Multiple `rate-limit` filters can be declared at the same scope or across service and route
scopes — each runs as an independent instance with its own bucket store. See
[Stacking the same policy multiple times](../policies.md#stacking-the-same-policy-multiple-times)
in the policies index for the full rules and examples.

### About `max-buckets`

Multi-key variants keep one bucket per key, so the number of buckets grows with your traffic's
diversity. `max-buckets` bounds that memory. When it's full, the least-recently-used buckets
are evicted.

**An evicted client starts fresh with a full bucket.** Don't set `max-buckets` so low that your
active clients are constantly evicted — every eviction hands that client a brand-new burst
allowance. Size it comfortably above your realistic count of distinct clients, paths, or
header values.

---

## Trying it out

Send requests faster than the bucket refills and watch them start being rejected:

```bash
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/echo
done
```

Once the bucket empties, the `200`s turn into `429`s.

## Related

- [IP restriction](ip-restriction.md) — to block outright rather than throttle
- [Token auth](token-auth.md) — required upstream of `key-from="client-id"`
- [Policies index](../policies.md)
