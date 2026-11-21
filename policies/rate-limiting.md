# Rate limiting

Caps how fast requests are accepted. A single `rate-limit` filter handles all rate-limiting
scenarios — `key-from` selects what gets its own budget:

| `key-from` | One budget per… | Use it to… |
|---|---|---|
| `"source-ip"` | client IP address | Stop one client overwhelming you |
| `"uri-group"` | everything matching a pattern | Cap total traffic to an endpoint group |
| `"uri"` | individual URI path | Stop one hot path starving the others |
| `"client-id"` | authenticated client | Per-user/per-tenant limiting (requires `token-auth` before it; Flow refuses the configuration otherwise, see [Order checks](../policies.md#order-checks-when-the-configuration-loads)) |
| `"header:<name>"` | value of a named header | Limit by tenant ID header or similar |
| `"composite"` | combination of fields | Per-client-per-route or other multi-dimensional limits |
| `"jwt-claim:<name>"` | authenticated client, with per-tier params | Subscription-tier limits — each tier gets its own params |

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

### A new client starts with a full bucket

Buckets are not created when Flow starts. There is one bucket per key — per client IP, per
authenticated client, per header value, depending on `key-from` — and each is created the
first time that key is seen, **already full**.

So the first request from a client it has never seen does not find an empty bucket and wait.
It finds `max-tokens` tokens and can spend all of them straight away. With
`max-tokens=1000 refill-qty=100 refill-interval-ms=100`, a brand-new client can send 1000
requests as fast as it can open connections, and only then settles to the sustained 1000/s.

Size `max-tokens` to the burst you are willing to absorb from someone arriving for the first
time, not just to the steady rate you want to allow.

### Refills are incremental, never a reset

Once a bucket is drained, it does not jump back to full after a quiet period. It regains
exactly `refill-qty` tokens every `refill-interval-ms`, up to the `max-tokens` ceiling.

With `refill-qty=1 refill-interval-ms=60000`, a client that has exhausted its bucket gets one
token back after a minute — one request, not a fresh allowance. Returning to a full bucket
takes `max-tokens ÷ refill-qty` intervals of complete silence.

This is what makes the limit hold under sustained pressure: a client cannot recover its burst
capacity by pausing briefly between batches.

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

### Three behaviors in one filter

The token bucket produces three distinct behaviors that have their own names in API gateway literature:

**Traffic throttle** — the sustained rate cap. Once the bucket is empty, only `refill-qty` requests pass per `refill-interval-ms`. This is the long-run ceiling: `refill-qty ÷ refill-interval-ms × 1000 req/s`.

**Burst control** — the short-run headroom. A client with a full bucket can send up to `max-tokens` requests back-to-back before throttling begins. `max-tokens` sets how large a burst is allowed before the sustained rate takes over.

**Immediate rejection** — when the bucket is empty, requests are rejected with `429` right away. Flow does not queue or delay requests. There is no minimum inter-request spacing enforced; requests either pass immediately or are rejected immediately. Some gateways call their bucket-empty behavior "spike arrest" and queue requests at a minimum rate — Flow does not do this. If you need queuing, it must happen at the client.

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

## `key-from="jwt-claim:<name>"` — subscription tier limits

**One bucket per authenticated client, with per-tier token-bucket parameters.** Each client's
tier is read from their JWT by `<name>` — a top-level claim name or a JSON Pointer path
(e.g. `"jwt-claim:/meta/tier"`). The bucket parameters come from the named tier in the
`tiers { }` block, or fall through to filter-level defaults for clients whose tier is absent
or unknown.

**Requirements:**
- `token-auth` must appear **before** `rate-limit` in `request-filters`.
- `expose-claims true` must be set on that `token-auth` filter.

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }

    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
        max-buckets=100000 threads=8 missing-tier="reject" {
        tiers {
            tier name="pro"   max-tokens=1000 refill-qty=100 refill-interval-ms=100
            tier name="basic" max-tokens=100  refill-qty=10  refill-interval-ms=1000
            tier name="trial" max-tokens=10   refill-qty=1   refill-interval-ms=1000
        }
    }
}
```

A client whose JWT carries `"sub_tier": "pro"` gets 1000-token buckets at 1000 req/s. A
client with `"sub_tier": "trial"` gets 10 req/s. The bucket key includes both the client ID
and the tier name, so a tier upgrade automatically creates a fresh bucket with the new limits.

**Missing tier behavior** is controlled by `missing-tier`:

| `missing-tier` | When the tier claim cannot be read | Default? |
|---|---|---|
| `"reject"` | Reject with `400 Bad Request` | Yes |
| `"skip"` | Pass through without rate limiting | — |

Use `missing-tier-status=<code>` to change the rejection status code — unlike the sibling
`missing-key-status` (for `header:<name>`/`composite`), this one is not currently range-checked
at config load, so pick a real 4xx code deliberately.

**Combine with `override-claims`** to let enterprise customers override their tier's params:

```kdl
filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
    max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
    max-buckets=100000 threads=8 missing-tier="reject" {
    tiers {
        tier name="pro"   max-tokens=1000 refill-qty=100 refill-interval-ms=100
        tier name="basic" max-tokens=100  refill-qty=10  refill-interval-ms=1000
    }
    override-claims {
        claim max-tokens-claim="rl_max_tokens" max-tokens-ceiling=10000
    }
}
```

Precedence (highest to lowest): JWT override claim → tier map → filter-level defaults.

### Worked examples: the token, and the config that reads it

The claim name in `key-from` must match what your identity provider actually puts in the
token. Each example below shows the JWT payload first, then the two filters that read it.

`expose-claims=true` is required on `token-auth` in every case — without it `rate-limit` sees
no claims and every request falls to `missing-tier`.

#### 1. A flat claim

The common case: a top-level claim holding the tier name.

```json
{
  "sub": "user-8471",
  "iss": "https://auth.example.com/",
  "exp": 1793452800,
  "sub_tier": "pro"
}
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
        max-buckets=100000 threads=8 missing-tier="reject" {
        tiers {
            tier name="pro"   max-tokens=1000 refill-qty=100 refill-interval-ms=100
            tier name="basic" max-tokens=100  refill-qty=10  refill-interval-ms=1000
        }
    }
}
```

The claim value `"pro"` selects `tier name="pro"`. The names must match exactly — matching is
case-sensitive.

#### 2. A nested claim

When the tier sits inside an object, use a JSON Pointer: a leading `/`, then one segment per
level.

```json
{
  "sub": "user-8471",
  "meta": {
    "billing": { "tier": "enterprise" }
  }
}
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:/meta/billing/tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 missing-tier="reject" {
        tiers {
            tier name="enterprise" max-tokens=5000 refill-qty=500 refill-interval-ms=100
        }
    }
}
```

The leading `/` is what makes it a path. Without it the whole string is treated as one literal
claim name, which is exactly what the next example relies on.

#### 3. A namespaced claim (Auth0, Okta and similar)

Providers that namespace custom claims produce names containing slashes. Because the name does
**not** start with `/`, Flow reads it as a literal key — no escaping needed.

```json
{
  "sub": "user-8471",
  "https://myapp.example.com/tier": "pro"
}
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:https://myapp.example.com/tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 missing-tier="reject" {
        tiers {
            tier name="pro" max-tokens=1000 refill-qty=100 refill-interval-ms=100
        }
    }
}
```

If a *nested* key itself contains a `/`, escape it as `~1` inside a pointer: `/org~1div/tier`
reads `claims["org/div"]["tier"]`. A literal `~` is `~0`.

#### 4. A numeric tier

Numbers are accepted and compared as their string form, so `tier name=` must be quoted.

```json
{ "sub": "user-8471", "plan_level": 2 }
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:plan_level" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 missing-tier="reject" {
        tiers {
            tier name="1" max-tokens=100  refill-qty=10  refill-interval-ms=1000
            tier name="2" max-tokens=1000 refill-qty=100 refill-interval-ms=100
        }
    }
}
```

Only strings and numbers resolve. A boolean, `null`, an object or an array resolves to nothing
and the request falls to `missing-tier`.

#### 5. One customer with limits of their own

`override-claims` lets a specific customer carry their own numbers in the token, bounded by a
ceiling you set, while everyone else uses the tier map.

```json
{
  "sub": "acme-corp",
  "sub_tier": "pro",
  "rl_max_tokens": 8000,
  "rl_refill_qty": 800
}
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
        max-buckets=100000 threads=8 missing-tier="reject" {
        tiers {
            tier name="pro"   max-tokens=1000 refill-qty=100 refill-interval-ms=100
            tier name="basic" max-tokens=100  refill-qty=10  refill-interval-ms=1000
        }
        override-claims {
            claim max-tokens-claim="rl_max_tokens" max-tokens-ceiling=10000 \
                  refill-qty-claim="rl_refill_qty" refill-qty-ceiling=1000
        }
    }
}
```

This token gets 8000 tokens refilling 800 at a time — its own values, under the ceilings.
A token asking for `20000` would be capped at the `10000` ceiling. A `pro` token with no
override claims gets the tier's 1000/100.

The ceilings are what stop a forged or over-generous token from granting unlimited traffic, so
set them whenever you enable `override-claims`.

#### 6. The same, with the overrides nested

Providers often group entitlements into an object rather than putting them at the top level.
`max-tokens-claim` and `refill-qty-claim` take a JSON Pointer for exactly this — the same
syntax `key-from` uses.

```json
{
  "sub": "acme-corp",
  "sub_tier": "pro",
  "limits": {
    "rl": {
      "max_tokens": 8000,
      "refill_qty": 800
    }
  }
}
```

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
        max-buckets=100000 threads=8 missing-tier="reject" {
        tiers {
            tier name="pro"   max-tokens=1000 refill-qty=100 refill-interval-ms=100
            tier name="basic" max-tokens=100  refill-qty=10  refill-interval-ms=1000
        }
        override-claims {
            claim max-tokens-claim="/limits/rl/max_tokens" max-tokens-ceiling=10000 \
                  refill-qty-claim="/limits/rl/refill_qty" refill-qty-ceiling=1000
        }
    }
}
```

Same result as the flat version: 8000 tokens refilling 800 at a time, under the ceilings. The
leading `/` is what makes the value a path — without it the whole string, slashes and all, is
treated as one literal claim name.

The tier claim and the override claims are resolved independently, so they can sit at
different depths: a top-level `sub_tier` alongside nested `limits.rl.*`, as above.

#### 7. Tokens with no tier at all

Decide what happens when the tier claim **cannot be read** — it is absent from the token, or
holds something that is not a string or number.

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    // Refuse the request — the default, with the status code of your choice.
    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 \
        missing-tier="reject" missing-tier-status=403 {
        tiers {
            tier name="pro" max-tokens=1000 refill-qty=100 refill-interval-ms=100
        }
    }
}
```

Or let it through ungoverned:

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }
    filter kind="rate-limit" key-from="jwt-claim:sub_tier" \
        max-tokens=50 refill-qty=5 refill-interval-ms=1000 missing-tier="skip" {
        tiers {
            tier name="pro" max-tokens=1000 refill-qty=100 refill-interval-ms=100
        }
    }
}
```

`missing-tier="skip"` and `missing-tier-status` cannot be combined — a status has no meaning
when nothing is rejected, and the configuration is refused at load.

**A claim that reads fine but names an undefined tier is a different case.** It is *not*
rejected: the request falls through to the filter-level `max-tokens`, `refill-qty` and
`refill-interval-ms`. Set those deliberately — they are the limit anyone presenting an
unrecognised tier will get.

#### Getting the claim into the token

The claim is an ordinary custom claim, added by your identity provider: an Action in Auth0, a
protocol mapper in Keycloak, a pre-token-generation trigger in Cognito, or a claims-mapping
policy in Entra ID. Flow reads the token; it never issues one.

To try a configuration before wiring up a provider, mint a short-lived token with any JWT
library, sign it with a key your `token-auth` filter trusts, and include the claim exactly as
shown above.


---

## Per-client limits from JWT claims — `override-claims`

Any multi-key variant (`client-id`, `source-ip`, `header:<name>`, `composite`, `uri`) can
carry an optional `override-claims` block that lets the JWT token override individual
token-bucket parameters per client.

**When to use it:** enterprise or custom-contract customers where each client's burst
allowance or sustained rate is individually negotiated and carried in their JWT.

**Requirements:**
- `token-auth` must appear **before** `rate-limit` in `request-filters`.
- `expose-claims true` must be set on that `token-auth` filter — this is what makes
  the JWT claims available to the rate limiter.

```kdl
request-filters {
    filter kind="token-auth" expose-claims=true {
        jwks-uri "https://auth.example.com/.well-known/jwks.json"
    }

    filter kind="rate-limit" key-from="client-id" \
        max-tokens=100 refill-qty=10 refill-interval-ms=1000 \
        max-buckets=100000 threads=8 {
        override-claims {
            claim max-tokens-claim="rl_max_tokens" \
                  refill-qty-claim="rl_refill_qty" \
                  max-tokens-ceiling=50000 \
                  refill-qty-ceiling=5000
        }
    }
}
```

If a request's JWT carries `"rl_max_tokens": 8000`, that client gets a 8000-token bucket
instead of the filter-level default of 100. A client whose JWT carries no `rl_max_tokens`
claim gets the filter default.

**Nested claim paths (JSON Pointer)** — if the limit values live inside a nested object, use
an RFC 6901 JSON Pointer starting with `/`:

```kdl
// JWT: { "entitlements": { "rate": { "max_tokens": 8000, "refill_qty": 800 } } }
filter kind="rate-limit" key-from="client-id" \
    max-tokens=100 refill-qty=10 refill-interval-ms=1000 \
    max-buckets=100000 threads=8 {
    override-claims {
        claim max-tokens-claim="/entitlements/rate/max_tokens" \
              refill-qty-claim="/entitlements/rate/refill_qty" \
              max-tokens-ceiling=50000 \
              refill-qty-ceiling=5000
    }
}
```

**Claim path formats:**

| Value | Reads from |
|---|---|
| `"rl_max_tokens"` | top-level claim `jwt["rl_max_tokens"]` |
| `"/limits/max_tokens"` | nested claim `jwt["limits"]["max_tokens"]` (JSON Pointer, RFC 6901) |
| `"/entitlements/rate/max_tokens"` | deeply nested `jwt["entitlements"]["rate"]["max_tokens"]` |

**A nested override, end to end** — payload, `token-auth` and `rate-limit` together — is
worked through in [example 6 above](#6-the-same-with-the-overrides-nested).

**The value must be a whole number.** It may be a JSON number or a string holding one:
`8000` and `"8000"` both work. Anything else — a decimal such as `8000.5`, a boolean, `null`,
an object or an array — **does not**.

**A value Flow cannot read is ignored silently.** There is no error, no warning and no
rejected request: the override is simply skipped and the client falls back to its tier
parameters, or to the filter defaults if it has no tier. A token carrying
`"rl_max_tokens": 8000.5` is therefore indistinguishable at runtime from one carrying no
override claim at all. If an override appears not to apply, check the value's **type** in the
token before changing the configuration.

**Ceiling enforcement:** `max-tokens-ceiling` and `refill-qty-ceiling` are hard limits set
in config. A JWT claim value above the ceiling is silently clamped. Always set ceilings in
production — without them a compromised token or auth server misconfiguration could grant
unrestricted burst to a client.

**Partial override:** each parameter is independent. A JWT carrying only `rl_max_tokens`
changes the burst ceiling; `refill-qty` comes from the filter default. A JWT carrying
neither gets both defaults.

**`refill-interval-ms` is not JWT-controllable** — it is always taken from the filter
config. This keeps ops in control of the timing tick; the JWT controls how large the bucket
is and how fast it refills per tick.

`uri-group` (shared single bucket) does not support `override-claims` — per-client
parameter overrides require per-client buckets. Use `client-id` for per-user limits from
JWT claims.

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
| `jwt-claim:<name>` | Tier claim absent, or not a string or number | **Reject `400 Bad Request`** (default) | Yes — `missing-tier`, `missing-tier-status` |

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
| `max-tokens` | all | Bucket size and initial fill — the burst allowance (filter-level default) |
| `refill-qty` | all | Tokens added per refill interval (filter-level default) |
| `refill-interval-ms` | all | Refill period in milliseconds (filter-level default) |
| `pattern` | `uri-group`, `uri` | Regex — filter only applies to paths that match |
| `max-buckets` | all multi-key variants | Maximum number of live buckets in memory |
| `threads` | all multi-key variants | Bucket cache shard count — set to match `threads-per-service` |
| `missing-key` | `header:<name>`, `composite` | What to do when the key field is absent: `"reject"` (default) or `"skip"` |
| `missing-key-status` | `header:<name>`, `composite` | HTTP 4xx status code (400–499) returned when rejecting a missing key; default `400`. Only valid with `missing-key="reject"`. |
| `tiers { tier name=... ... }` | `jwt-claim:<name>` | Named tier definitions — each `tier` row sets `max-tokens`, `refill-qty`, `refill-interval-ms` for one tier claim value |
| `missing-tier` | `jwt-claim:<name>` | What to do when the tier claim cannot be read — absent, or not a string or number: `"reject"` (default) or `"skip"`. A claim that reads fine but names an undefined tier is **not** covered by this; it falls through to the filter-level defaults |
| `missing-tier-status` | `jwt-claim:<name>` | Status code returned for `missing-tier="reject"`; default `400`. Not range-checked at config load (unlike `missing-key-status`'s enforced 400–499) — any integer is accepted, so pick a real 4xx code deliberately. |
| `override-claims { claim ... }` | all multi-key variants | JWT claim–driven per-client parameter overrides — see [Per-client limits from JWT claims](#per-client-limits-from-jwt-claims--override-claims) |
| `max-tokens-claim` | inside `override-claims` | Claim path (flat or JSON Pointer) for `max-tokens` override |
| `refill-qty-claim` | inside `override-claims` | Claim path (flat or JSON Pointer) for `refill-qty` override |
| `max-tokens-ceiling` | inside `override-claims` | Hard ceiling on JWT-supplied `max-tokens` |
| `refill-qty-ceiling` | inside `override-claims` | Hard ceiling on JWT-supplied `refill-qty` |

`uri-group` uses one shared bucket — `max-buckets` and `threads` are not needed.
`client-id` hardcodes `401` for a missing identity — `missing-key` and `missing-key-status` do not apply.
`override-claims` on `uri-group` has no effect — use `jwt-claim:` or `client-id` for per-client JWT-driven limits.

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
