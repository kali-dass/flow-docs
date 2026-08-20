# JWT validation

Verifies a JSON Web Token (JWT) on incoming requests before they reach your backend. Supports
multiple issuers on one route, JWTs verified against a live JWKS endpoint *or* a key pinned to
a local file, and enforces algorithm/audience/expiry/custom-claim rules.

**Phase:** `request-filters`

> **Pre-release note.** `jwt-validate` is new. Config syntax and defaults may still change —
> see the pre-release notice at the top of the [documentation home](../README.md).

## Quick example

```kdl
path-control {
    request-filters {
        filter kind="jwt-validate" {
            issuers {
                issuer name="https://idp.example.com" \
                    jwks-uri="https://idp.example.com/.well-known/jwks.json" \
                    algorithm="RS256"
            }
        }
    }
}
```

This is the minimum viable config: one issuer, keys fetched live from a JWKS endpoint and kept
fresh automatically. A request without a valid `Bearer` token in `Authorization` is rejected
with `401`; a valid one proceeds to your backend.

## How it works, in order

1. **Extract** the token from wherever you've configured (`Authorization` header by default).
2. **Peek** the token's `iss` claim — *without* verifying the signature yet — to pick which
   configured issuer's keys to check against.
3. **Resolve the key** — for a JWKS issuer, look up by the token's `kid`; for a static-key
   issuer, there's only one key.
4. **Verify the signature**, using the algorithm *you configured* for that issuer — never
   whatever the token itself claims (see [Why algorithm is pinned](#why-algorithm-is-pinned)).
5. **Check claims** — `exp`, `nbf`, `iss` (for real this time), `aud` if configured, and any
   `required-claims`.
6. On success, the token's claims become available to policies that run later in the request
   (subject, issuer, and the full claim set).

A request that fails any step gets `401` (`invalid_token`, `invalid_request`, or plain
`Unauthorized` depending on which check failed — see [Responses](#responses)) or, if the
*issuer's key source itself* is unavailable, `503`.

---

## Choosing where the token comes from

Exactly **one** of these three — set more than one and the filter fails to load.

### `token-header-names` (default)

```kdl
filter kind="jwt-validate" token-header-names="authorization" { issuers { ... } }
```

Comma-separate for more than one header:

```kdl
filter kind="jwt-validate" token-header-names="authorization,x-api-token" { issuers { ... } }
```

**This isn't a preference order (first found wins) — every listed header is checked, and the
results must agree.** Flow reads all of them, strips the scheme from each, and collects the
*distinct* values found:

- Token present in exactly one of the listed headers → that's the token, whichever header it
  came from.
- Same token value repeated in more than one of the listed headers → still just one distinct
  value, so it's used normally.
- **Different** token values across two of the listed headers → rejected (`401`,
  `invalid_request`) as an ambiguous credential — Flow won't guess which one to trust.
- Present in none of them → `401` (missing).

Listing more than one header name is for accepting a token from *either* of two places a client
might put it (e.g. a legacy `x-api-token` header alongside standard `authorization`), not for
giving one header priority over another.

If you don't set any of the three source options, this is what you get, with
`token-header-names="authorization"`. Pairs with [`token-scheme`](#token-scheme).

### `token-cookie-names`

```kdl
filter kind="jwt-validate" token-cookie-names="session_jwt" { issuers { ... } }
```

For browser-session-style auth where the token lives in a cookie instead of a header.

> **CSRF caveat.** A cookie is sent automatically by the browser on cross-site requests too — a
> header requires JavaScript to attach it explicitly, so it can't be forged by a plain
> cross-site request the way a cookie can. If you use cookie-sourced tokens, your deployment
> needs its own CSRF mitigation (`SameSite` cookie attribute, a synchronizer token); Flow has no
> way to verify a cookie-carried request wasn't cross-site-forged.

### `token-query-names`

```kdl
filter kind="jwt-validate" token-query-names="jwt" { issuers { ... } }
```

For clients that can't set headers or cookies at all.

> **Leakage caveat.** A token in the query string ends up in server access logs, browser
> history, and the `Referer` header sent to any third-party resource the page subsequently
> loads. Prefer a header if you have any choice in the matter.

### `token-scheme`

Only valid alongside `token-header-names` (cookies and query params carry the bare token, no
scheme prefix — setting this with either of those is rejected).

```kdl
filter kind="jwt-validate" token-header-names="authorization" token-scheme="Bearer" { issuers { ... } }
```

`Bearer` is the default — you don't need to set it for the standard case. The match is
case-insensitive (`bearer`, `BEARER`, `Bearer` all work), per RFC 7235.

**To accept a bare token with no scheme prefix at all** (some non-standard APIs put the raw
JWT directly in `Authorization`, with no `Bearer ` in front):

```kdl
filter kind="jwt-validate" token-header-names="authorization" token-scheme="none" { issuers { ... } }
```

`token-scheme=""` (empty string) is rejected — it's ambiguous whether that's a deliberate
"no scheme" or a stray empty value, so the explicit `"none"` keyword is required instead.

---

## Claim checks (apply to every issuer)

### `leeway-secs`

Clock-skew tolerance for `exp`/`nbf`. If your Flow instance and your IdP's clock can drift by a
few seconds, this widens the boundary so a token isn't rejected for being technically a few
seconds early/late.

```kdl
filter kind="jwt-validate" leeway-secs=30 { issuers { ... } }
```

Default `0`. Capped at `300` (5 minutes) — a larger value is rejected, so a typo can't turn
"always verify expiry" into "expiry effectively disabled".

### `max-expiration-secs`

Caps how much **validity a token has left**, checked fresh on every single request — not how
long the token was originally valid for at issuance. The two are easy to conflate, so it's
worth spelling out the formula and a worked timeline.

**The check**: on each request, Flow computes `remaining = token's exp claim − current time`.
If `remaining > max-expiration-secs`, the token is rejected (`401`), even though it is
otherwise completely valid (good signature, not yet expired by its own `exp`). The token's
`exp` claim itself is never modified — this is a second, independent ceiling Flow enforces on
top of it.

**Why this exists**: this setting doesn't control tokens *you* issue — it constrains tokens
issued by an upstream IdP you don't control. If that IdP mints tokens with a 30-day lifetime and
one leaks (log, browser history, compromised client), the token stays honored for however much
of those 30 days remain — Flow can't revoke it and the IdP may not offer a revocation
mechanism. `max-expiration-secs` puts a hard ceiling on how long *Flow* will still accept a
long-lived token, without needing anything from the IdP.

**Worked example** — a token with a 24-hour lifetime, `max-expiration-secs=3600` (1 hour):

```kdl
filter kind="jwt-validate" max-expiration-secs=3600 { issuers { ... } }
```

| Token issued at | Token's own `exp` | Request arrives at | Remaining (`exp − now`) | Result |
|---|---|---|---|---|
| 09:00 | 09:00 + 24h = next day 09:00 | 09:05 (5 min after issuance) | ~23h 55m | **Rejected** — 23h 55m > 1h cap |
| 09:00 | next day 09:00 | next day 08:00 (23h later) | 1h | **Accepted** — exactly at the cap |
| 09:00 | next day 09:00 | next day 08:30 (23.5h later) | 30 min | **Accepted** — 30 min ≤ 1h cap |
| 09:00 | next day 09:00 | next day 09:30 (past `exp`) | negative | **Rejected** — already expired by its own `exp`, checked separately |

Notice the same physical token is rejected at 09:05 and accepted at 08:00 the next day — its
`exp` claim never changes, but how much of it is left keeps shrinking as time passes, and it
only clears the cap once there's an hour or less of validity remaining. In effect, a 24-hour
token behaves, from Flow's point of view, like it's only usable during its **last hour** of
life.

**To disable the cap explicitly** (accept a token with any amount of remaining validity, as
long as it hasn't hit its own `exp`):

```kdl
filter kind="jwt-validate" max-expiration-secs="none" { issuers { ... } }
```

This is also the default if you don't set `max-expiration-secs` at all — most deployments where
you control the IdP and already mint short-lived tokens don't need this cap.

`max-expiration-secs=0` is rejected rather than treated as "disabled" — `0` would literally mean
"reject every token, since every token has at least some positive amount of remaining validity
the instant it's issued," which isn't a usable value someone would actually want; use `"none"`
to disable the check instead.

### `required-claims`

**Presence check only — it does not check what value the claim holds.** This lists claim
*names* that must exist in the token (and not be JSON `null`), beyond the standard ones Flow
already checks (`exp`, `iss`, and `aud` when configured). It does **not** let you require a
specific value, like "the `scope` claim must equal `abc`" or "must contain `abc`" — any
non-null value passes, whatever it is.

```kdl
filter kind="jwt-validate" required-claims="scope,tenant_id" { issuers { ... } }
```

With this config:

| Token's claims | Result | Why |
|---|---|---|
| `{"scope": "read:orders", "tenant_id": "acme"}` | **Accepted** | Both claims present and non-null — their actual values are irrelevant |
| `{"scope": "", "tenant_id": "acme"}` | **Accepted** | `scope` is present and non-null — an empty string still passes; `required-claims` doesn't check for a non-empty or specific value |
| `{"scope": null, "tenant_id": "acme"}` | **Rejected** | `scope` is present but explicitly `null` |
| `{"tenant_id": "acme"}` (no `scope` key at all) | **Rejected** | `scope` is missing entirely |
| `{"scope": "read:orders"}` (no `tenant_id`) | **Rejected** | `tenant_id` is missing |

Comma-separated for more than one — every listed claim must independently pass. Default: none
required.

**If you need to enforce a specific value** (e.g. "the `scope` claim must equal `abc`", or "must
contain a particular value"), `required-claims` can't do that today — there's no per-claim
value-matching in this filter. The one exception is `aud` (audience), which does get
value-matching via the separate [`audience`](#audience) setting on each issuer.

### `skip-preflight`

```kdl
filter kind="jwt-validate" skip-preflight=true { issuers { ... } }
```

When `true`, a browser CORS preflight request (`OPTIONS` carrying
`Access-Control-Request-Method`) bypasses this filter entirely — no token required. Browsers
generate preflight requests themselves and can never attach a bearer token to them, so without
this, every authenticated call from a browser client would fail its own preflight check before
the real request is even sent.

Default `false` (preflights are validated like any other request). If you're serving a
browser-based client and requests mysteriously never reach your API, this is usually why —
enable it.

---

## JWKS behavior (applies to every JWKS-sourced issuer in this filter)

These control how Flow fetches and refreshes keys for any issuer using `jwks-uri` (not
static-key issuers — they have no fetch to schedule).

### `jwks-refresh-secs`

How often Flow re-fetches the JWKS document on its normal schedule.

```kdl
filter kind="jwt-validate" jwks-refresh-secs=300 { issuers { ... } }
```

Default `300` (5 minutes).

### `jwks-min-refresh-secs`

A token can arrive carrying a `kid` Flow doesn't currently have — usually because the IdP just
rotated keys and Flow hasn't caught up yet. When that happens, Flow can trigger an out-of-band
refresh *before* the next scheduled one, so a real rotation resolves faster than
`jwks-refresh-secs` would allow. This setting is the floor between such early refreshes, so a
flood of requests carrying bogus `kid`s can't turn into a flood of requests at your IdP.

```kdl
filter kind="jwt-validate" jwks-min-refresh-secs=180 { issuers { ... } }
```

Default `180` (3 minutes) — smaller than `jwks-refresh-secs`'s default, so a genuine rotation
typically resolves in under 3 minutes instead of waiting up to 5.

### `jwks-max-stale-secs`

If Flow can't reach the JWKS endpoint, it keeps serving the last keys it successfully fetched —
up to this many seconds past that last success. Past this window, Flow stops trusting those
keys (see [Staleness → 503](#staleness--503)).

```kdl
filter kind="jwt-validate" jwks-max-stale-secs=3600 { issuers { ... } }
```

Default `3600` (1 hour). Longer tolerates a longer IdP outage without disrupting your traffic;
shorter makes key revocation take effect faster if the IdP ever needs to pull a compromised key.
There's no value that's simultaneously "most available" and "revokes fastest" — pick based on
which matters more for your deployment.

### `jwks-required-at-startup`

```kdl
filter kind="jwt-validate" jwks-required-at-startup=false { issuers { ... } }
```

Default `true`: if the *very first* JWKS fetch (at Flow startup) fails for any issuer, Flow
refuses to start — a gateway that won't boot is easier to notice than one that boots and
silently 401s every request needing that issuer.

Set to `false` only if Flow and your IdP genuinely start up together in the same rollout (e.g.
both are new containers in one deployment) and the IdP might not be reachable in Flow's first
few seconds. With `false`, Flow boots anyway; requests needing that issuer get `503` until the
first successful background fetch completes.

### `jwks-verify-cache-size`

Bounds how many recently-verified tokens Flow remembers, so a repeat request with the same
token skips full signature verification. This exists because RSA/ECDSA verification is
measurably expensive at high request rates — the cache is what keeps that cost off your
hot path for repeat traffic.

```kdl
filter kind="jwt-validate" jwks-verify-cache-size=10000 { issuers { ... } }
```

Default `10000` entries. This is a cap on *entries*, not bytes. `0` is rejected — the
verification cache can't be disabled (see the note in [Performance notes](#performance-notes)).

---

## Per-issuer settings

Everything above is filter-wide. These live inside each `issuer { }` entry — a filter can have
more than one issuer, and each can independently use JWKS or a static key.

```kdl
filter kind="jwt-validate" {
    issuers {
        issuer name="https://idp-a.example.com" jwks-uri="https://idp-a.example.com/keys" algorithm="RS256"
        issuer name="https://idp-b.example.com" hmac-secret-path="/etc/flow/keys/b.secret" algorithm="HS256"
    }
}
```

A token is routed to the matching issuer by its (unverified) `iss` claim, then fully verified
against that issuer's own key and algorithm. This lets one route accept tokens from several
identity providers, each with its own key material.

### `name`

```kdl
issuer name="https://idp.example.com" ...
```

**Required.** Matched against the token's `iss` claim — must be unique among the issuers in one
filter (it's how Flow decides which issuer's key to check a given token against). Can't be
empty or whitespace-only.

### `algorithm`

```kdl
issuer name="..." algorithm="RS256" ...
```

**Required.** One of `HS256`, `HS384`, `HS512`, `RS256`, `RS384`, `RS512`, `PS256`, `PS384`,
`PS512`, `ES256`, `ES384`, `EdDSA`.

This is **always** taken from here — never read from the token's own header, and never
inferred from the key. See [Why algorithm is pinned](#why-algorithm-is-pinned) for why that
matters.

### Choosing a key source: `jwks-uri`, `public-key-path`, or `hmac-secret-path`

Exactly **one** of these three per issuer.

**`jwks-uri`** — fetch keys live from a JWKS endpoint, kept fresh automatically:

```kdl
issuer name="https://idp.example.com" jwks-uri="https://idp.example.com/.well-known/jwks.json" algorithm="RS256"
```

Recommended for most setups — supports key rotation without touching Flow's config. Requires
`algorithm` to be one that matches how the endpoint's keys are meant to be used (Flow doesn't
infer this from the fetched key either).

**`public-key-path`** — a PEM-encoded public key file, loaded once at startup, for an
asymmetric algorithm (`RS*`/`PS*`/`ES*`/`EdDSA`):

```kdl
issuer name="https://idp.example.com" public-key-path="/etc/flow/keys/idp-pub.pem" algorithm="RS256"
```

**`hmac-secret-path`** — a raw secret file, loaded once at startup, for an HMAC algorithm
(`HS256`/`HS384`/`HS512`):

```kdl
issuer name="https://idp.example.com" hmac-secret-path="/etc/flow/keys/shared.secret" algorithm="HS256"
```

Use a static key source when the issuer doesn't publish JWKS at all, or when you'd rather pin
one specific key file than trust a live endpoint. The tradeoff: a static key never rotates
without a config change and a reload — there's no fetch loop to pick up a new key
automatically the way `jwks-uri` does.

`public-key-path` with an HMAC algorithm (or `hmac-secret-path` with an asymmetric one) is
rejected at config load — the mismatch is caught before Flow ever tries to use the file, not
left to fail cryptically at request time.

### `jwks-ca-cert-path`

Only valid alongside `jwks-uri` (rejected if set on a static-key issuer — there's no TLS fetch
to apply it to).

```kdl
issuer name="https://idp.example.com" jwks-uri="https://idp.internal.example.com/keys" \
    jwks-ca-cert-path="/etc/flow/certs/internal-ca.pem" algorithm="RS256"
```

CA bundle to trust when fetching this issuer's JWKS over TLS. Only needed for an internal IdP
whose certificate isn't signed by a public CA — a public IdP (Auth0, Okta, your cloud
provider's identity service) works with this left unset, using the platform's default trust
store.

### `audience`

Comma-separating more than one value here means **"accept the token if it's meant for *any* of
these audiences"** — an OR match, not a requirement that every listed value be present.

```kdl
issuer name="https://idp.example.com" jwks-uri="..." algorithm="RS256" audience="api.example.com,api-internal.example.com"
```

**How the match works**: a JWT's `aud` claim (per RFC 7519 §4.1.3) can itself be either a single
string or an array of strings — a token can legitimately be issued for more than one audience.
Flow's check passes if there is **any overlap** between your configured list and whatever the
token's `aud` claim contains — checked as sets, not compared as an ordered/exact list.

| `audience` config | Token's `aud` claim | Result |
|---|---|---|
| `"api.example.com"` | `"api.example.com"` | **Accepted** — exact match |
| `"api.example.com,api-internal.example.com"` | `"api-internal.example.com"` | **Accepted** — token matches the second listed value |
| `"api.example.com,api-internal.example.com"` | `["api-internal.example.com", "some-other-service"]` (array) | **Accepted** — `api-internal.example.com` is common to both sides |
| `"api.example.com,api-internal.example.com"` | `"billing.example.com"` | **Rejected** — no overlap with either configured value |
| `"api.example.com"` | *(no `aud` claim in the token at all)* | **Rejected** — `audience` configured but claim absent |

So `audience="api.example.com,api-internal.example.com"` reads as "this route accepts tokens
scoped to `api.example.com` *or* `api-internal.example.com`" — useful when the same issuer mints
tokens for more than one downstream service and you want this route to accept tokens meant for
either. It does **not** mean the token must list both audiences at once.

If `audience` is left unset entirely, `aud` isn't checked at all for this issuer — an explicit
choice, not an oversight; leave it off only if you genuinely don't need audience scoping.

---

## Multiple issuers sharing one `jwks-uri`

If two different `jwt-validate` filters — even on different routes or services — configure an
issuer with the **same** `jwks-uri`, Flow shares one JWKS store and one fetch/refresh loop
between them rather than running two independently. Sensible, since they're both talking about
the same key material.

Because of that sharing, both configurations must **agree**: same `algorithm`,
`jwks-ca-cert-path`, `jwks-required-at-startup`, `jwks-refresh-secs`, `jwks-min-refresh-secs`,
and `jwks-max-stale-secs`. A mismatch is rejected at config load, naming which issuer/filter
hit the conflict — not silently resolved by whichever filter happened to load first.

---

## Why algorithm is pinned

`algorithm` on each issuer is never read from the token's own header, and never inferred from
the key material. This is a deliberate defense against the **algorithm-confusion attack**: an
attacker takes a service's RSA *public* key (which is, by design, public) and uses it as an
HMAC *secret* to forge a token, re-signed with `HS256` instead of the RSA algorithm the service
actually expects. If a verifier trusted the token's own claimed algorithm, this forged token
would pass. Flow always verifies against the algorithm **you configured**, so a token claiming
a different algorithm than expected is rejected outright, regardless of what key material it
was signed with.

## Responses

| Situation | Status | `WWW-Authenticate` |
|---|---|---|
| No token found in the configured source | `401` | `Bearer realm="flow"` |
| More than one distinct token presented (e.g. different values in two configured header names) | `401` | `Bearer error="invalid_request"` |
| Malformed token, unrecognized issuer, bad signature, expired, wrong audience, missing required claim, `max-expiration-secs` exceeded, unresolvable `kid` | `401` | `Bearer error="invalid_token"` |
| The matched issuer's key source is stale or has never been successfully fetched | `503` | *(none)* |

Two things worth knowing:

- **Multiple distinct tokens are rejected, not resolved by picking one.** If a client (or an
  attacker) presents different token values across the sources you've configured names for
  (e.g. two different header names both set), Flow won't silently validate one and ignore the
  other — the whole request is rejected as ambiguous.
- **`401` reasons are deliberately not distinguished in the response body or status.** A
  malformed token and a token with a bad signature both come back as `error="invalid_token"`,
  "Bad token" — the *why* is only visible in Flow's own logs, so the response itself can't be
  used to probe which specific check failed.
- **`503` means Flow's problem, not yours.** If the matched issuer's JWKS store is stale (see
  `jwks-max-stale-secs`) or has never completed a fetch, Flow can't currently verify *any*
  token for that issuer — including one that would otherwise be perfectly valid. Retrying a
  request that hit a `503` may well succeed once the IdP is reachable again.

## Staleness → 503

A JWKS-sourced issuer's keys are treated as "too old to trust" once `jwks-max-stale-secs` has
passed since the last successful fetch — including if the *very first* fetch never succeeded.
When that happens, every request needing that issuer gets `503` (not `401`) until the next
successful refresh. This is deliberate: an unreachable IdP is Flow's problem to report clearly,
not something that should look like every one of your callers suddenly presented a bad
credential.

A static-key issuer is never stale — there's no fetch to have failed; the key was loaded once
at startup and is either usable (Flow started successfully) or the filter failed to load at all.

## Performance notes

Signature verification is measurably expensive for RSA/ECDSA algorithms — enough to be visible
in throughput at high request rates. The verification-result cache
(`jwks-verify-cache-size`) exists specifically to keep that cost off repeat requests from the
same client; it can't be turned off, since doing so would remove Flow's only mitigation for
this cost.

If your traffic is dominated by distinct, never-repeated tokens (unusual, but possible with
some client patterns), the cache won't help much and RS*/PS*/ES* algorithms will cost roughly
what raw signature verification costs per request. HMAC algorithms (`HS*`) are comparatively
cheap and less affected either way.

## The token reaches your backend unchanged

`jwt-validate` runs in the `request-filters` phase purely as a gate: it reads the token,
verifies it, and either rejects the request or lets it through. It never removes, rewrites, or
otherwise touches the header/cookie/query parameter the token came from. **Whatever the client
originally sent — the raw JWT, in the same place — is exactly what reaches your backend.** If
your backend independently needs to parse or re-verify the token, it can, since nothing about
it changes in transit through Flow.

## Not yet supported

**Forwarding *decoded claims* to your backend as separate headers is not implemented yet.**
There is currently no way to take a claim from the verified token (e.g. `sub`, `scope`) and
inject it as its own header (e.g. `X-User-Id`) on the request Flow sends upstream — only the
original, unparsed token passes through, as described above. If your backend wants
Flow-provided identity without re-parsing the JWT itself, that's not available yet; this is
planned, tracked in the roadmap alongside the other authentication work; see the
[pre-release notice](../README.md).

## Trying it out

Mint a test token (using [jwt.io](https://jwt.io) or a local script) signed with a secret,
point a static-key issuer at it:

```kdl
services {
    Test {
        listeners { "0.0.0.0:8080" }
        routes {
            route {
                match { path-prefix "/" }
                allowed-methods "GET"
                allowed-protocols "http"
                connectors { "127.0.0.1:3000" }
                path-control {
                    request-filters {
                        filter kind="jwt-validate" {
                            issuers {
                                issuer name="https://idp.example.com" \
                                    hmac-secret-path="/etc/flow/keys/test.secret" \
                                    algorithm="HS256"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

```bash
curl -H "Authorization: Bearer <your-test-token>" http://localhost:8080/
```

Without the header, or with a malformed/expired token, you'll get a `401` with a
`WWW-Authenticate` header explaining what was wrong.

## Related

- [Policies index](../policies.md)
- [Design decisions](../design-decisions.md) — choosing between JWKS and static keys
