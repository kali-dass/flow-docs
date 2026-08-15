# Rate limiting

Caps how fast requests are accepted. Flow offers three rate limiters, differing only in
**what gets its own budget**:

| Policy `kind` | One budget per… | Use it to… |
|---|---|---|
| `rate-limit-single-uri-group` | **everything** matching a pattern | Cap total traffic to an endpoint group |
| `rate-limit-multi-source-ip` | **client IP** | Stop one client overwhelming you |
| `rate-limit-multi-uri` | **URI path** | Stop one hot path starving the others |

**Phase:** `request-filters` (all three)

## How the limiter works

All three use a **token bucket**:

- The bucket holds up to `max-tokens`.
- Every request takes one token.
- The bucket refills by `refill-qty` every `refill-interval-ms`.
- When the bucket is empty, requests are **rejected**.

Two numbers fall out of that:

- **Sustained rate** = `refill-qty` ÷ `refill-interval-ms`
- **Burst** = `max-tokens` — how many requests can arrive at once before throttling begins

So a bucket with `max-tokens=50`, `refill-qty=5`, `refill-interval-ms=200` allows a burst of
50, then settles to **25 requests/second**.

Set `max-tokens` to the burst you're willing to absorb, and the refill pair to the rate you
want to sustain. They are independent — a large burst with a slow refill is fine, and is often
what you want for bursty clients.

---

## `rate-limit-single-uri-group`

**One shared bucket** for every request whose path matches `pattern`. All callers draw from the
same budget.

```kdl
request-filters {
    filter kind="rate-limit-single-uri-group" pattern="^/echo" \
        max-tokens=50 refill-qty=5 refill-interval-ms=200
}
```

Burst of 50, then **25 requests/second** shared across all clients.

Use it to protect a fragile backend from *total* load, regardless of who is calling.

---

## `rate-limit-multi-source-ip`

**One bucket per source IP.** Each client gets its own budget, so one noisy client cannot
starve the others.

```kdl
request-filters {
    filter kind="rate-limit-multi-source-ip" \
        max-tokens=100 refill-qty=10 refill-interval-ms=100 \
        max-buckets=50000 threads=8
}
```

Each IP gets a burst of 100 and **100 requests/second** sustained.

> ⚠️ **Check what "source IP" means in your deployment.** This is the source IP of the TCP
> connection. If Flow sits behind another proxy, a CDN, or a cloud load balancer, that will be
> the *proxy's* IP — so every client shares one bucket and the policy does the opposite of
> what you intended. Flow does not currently read `X-Forwarded-For`.

---

## `rate-limit-multi-uri`

**One bucket per unique URI path** matching `pattern`.

```kdl
request-filters {
    filter kind="rate-limit-multi-uri" pattern="^/echo" \
        max-tokens=20 refill-qty=2 refill-interval-ms=500 \
        max-buckets=10000 threads=8
}
```

Each distinct path under `/echo` gets a burst of 20 and **4 requests/second** sustained.

---

## Options

| Option | Applies to | Meaning |
|---|---|---|
| `max-tokens` | all | Bucket size — the burst allowance |
| `refill-qty` | all | Tokens added each interval |
| `refill-interval-ms` | all | How often tokens are added |
| `pattern` | `single-uri-group`, `multi-uri` | Regex selecting which paths the limiter applies to |
| `max-buckets` | `multi-*` | Cap on how many buckets are held in memory |
| `threads` | `multi-*` | Shard count for the bucket cache — match your `threads-per-service` |

### About `max-buckets`

The `multi-*` limiters keep one bucket per key (per IP, or per path), so the number of buckets
grows with your traffic's diversity. `max-buckets` bounds that memory. When it's full, the
least-recently-used buckets are evicted.

**An evicted client starts fresh with a full bucket.** So don't set `max-buckets` so low that
your active clients are constantly evicted — every eviction hands that client a brand-new burst
allowance, and the limit you configured stops being enforced. Size it comfortably above your
realistic count of distinct clients or paths.

---

## Trying it out

Send requests faster than the bucket refills and watch them start being rejected:

```bash
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/echo
done
```

Once the bucket empties, the `200`s turn into rejections.

## Related

- [Block CIDR range](block-cidr-range.md) — to block outright rather than throttle
- [Policies index](../policies.md)
