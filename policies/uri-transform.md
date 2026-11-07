# URI transform

Rewrites the path and query string of the request **before it reaches your backend**. The
client never sees this — its own request and the URL it holds are completely untouched.

**Phase:** `upstream-request` only

```kdl
path-control {
    upstream-request {
        filter kind="uri-transform" strip-prefix="/api/v1"
    }
}
```

This is the opposite direction from [Redirect](redirect.md): a redirect sends the *client*
somewhere else and never calls your backend at all; `uri-transform` changes what your
*backend* receives, while the client's own URL stays exactly as it was. They compose: a
`redirect` can send a client to a whole new API, and that new route can use `uri-transform` to
map the public path onto whatever your backend's real layout happens to be.

It is also deliberately headers-free: changing the upstream `Host` is
[Header transform](header-transform.md)'s job, not this policy's. `uri-transform` only ever
touches the path and the query string.

## Options

Exactly one of `uri`, `strip-prefix`, or a `regex-uri` block may be set — they are different
ways to produce the new path, so combining them is refused at load. `set-query-param` and
`remove-query-param` are independent of all three and may be combined with any of them.

| Option | Meaning | Multiple rows? |
|---|---|---|
| `uri` | A fixed path (and optional `?query`) sent to the backend for every request, for example `uri="/orders"`. Must start with `/`. | — (a single setting, not a block) |
| `strip-prefix` | Removes a path prefix at a `/` boundary, for example `strip-prefix="/api/v1"`. `/api/v1` removes the start of `/api/v1/orders` (leaving `/orders`) but not of `/api/v10/orders` (not a boundary) — a request that does not start with the prefix is forwarded unchanged. | — (a single setting, not a block) |
| `regex-uri` block | One `pattern`/`replacement` row, matched against the request's raw path and query together, substituting any capture groups (`$1`, `$2`, …) into `replacement`. | **No — exactly one row.** A second row inside the block is refused at load (see below). |
| `set-query-param` block | Adds a query parameter, or replaces it if already present. | **Yes** — one row per parameter you want to set. |
| `remove-query-param` block | Removes a query parameter by exact name. | **Yes** — one row per parameter you want to remove. |

At least one of the five must be set — an empty `uri-transform` filter is refused at load.

**`regex-uri` takes exactly one row — not a list of patterns.** This is refused at load:

```kdl
regex-uri {
    rule pattern="^/a/(.+)$" replacement="/x/$1"
    rule pattern="^/b/(.+)$" replacement="/y/$1"   # refused: a second row
}
```

If you need more than one rewrite rule, use several `uri-transform` filters, one per
`regex-uri`, in the order you want them to run (see "Why only one `regex-uri` pattern" below).

**`set-query-param` and `remove-query-param` each take as many rows as you need**, inside the
one block — not one block per parameter:

```kdl
filter kind="uri-transform" {
    set-query-param {
        param name="version" value="2"
        param name="source" value="gateway"
    }
    remove-query-param {
        param name="debug"
        param name="trace"
    }
}
```

This removes `debug` and `trace` from the query string and sets `version=2` and
`source=gateway`, all in one `uri-transform` filter.

The regex is matched against the exact bytes of the request's own path and query — never
Flow's internally normalized path, never re-encoded.

## Capture groups

Parentheses in `pattern` capture part of the match for use in `replacement`. Groups are numbered
left to right, starting at `1` (group `0` would be the whole match, but there's no need to write
it: the rest of `replacement` already covers whatever the pattern did not capture).

| Request | `pattern` | `replacement` | Backend sees |
|---|---|---|---|
| `/orders/42` | `^/orders/(.+)$` | `/v2/orders/$1` | `/orders/42` → `/v2/orders/42` |
| `/shop/widgets/99` | `^/shop/([^/]+)/([^/]+)$` | `/catalog/$2?category=$1` | `/shop/widgets/99` → `/catalog/99?category=widgets` |

**Writing `$1`, `$2`, …:** a reference is `$` followed by digits or letters, parsed as long as
possible — so `$1a` means a group *named* `1a`, not group `1` followed by the letter `a`. If you
need a digit or letter to follow a group number directly, wrap it in braces: `${1}a` is
unambiguously group `1` then the letter `a`. To write a literal `$` in `replacement`, use `$$`.

**Named groups** are also supported, written `(?<name>...)` in `pattern` and `$name` (or
`${name}`) in `replacement` — useful when a pattern has several groups and numbers alone would
be hard to keep track of:

```kdl
regex-uri {
    rule pattern="^/shop/(?<category>[^/]+)/(?<id>[^/]+)$" replacement="/catalog/$id?category=$category"
}
```

A group reference that did not participate in the match (for example, inside an alternation
branch that was not taken) is replaced with an empty string, not an error.

## Why only one `regex-uri` pattern

A list of pattern/replacement pairs (tried in order, first match wins) adds "which one
matched" ambiguity to both debugging and documentation. If one pattern genuinely is not enough
for your case, chain two `uri-transform` filters instead.

## Not supported here

- **Rewriting headers.** Use [Header transform](header-transform.md) — it is independently
  ordered in `upstream-request`, so you can combine a header change with a `uri-transform` in
  one block, in the order you write them.
- **Changing the `Host` header.** Same reason — `header-transform` already does this.
- **Rewriting the HTTP method.** Not offered; it has consequences for caching and for the
  `allowed-methods` check that already matched the route on the original method.
- **gRPC routes.** A gRPC call's path *is* the procedure name
  (`/package.Service/Method`) — rewriting it would dispatch a different method with an almost
  certainly incompatible request/response shape, not just change a URL. `uri-transform` is
  refused at load on any route with `allowed-protocols "grpc"` or `"grpcs"`.

## Common uses

**Strip an API version prefix your backend does not know about:**

```kdl
upstream-request {
    filter kind="uri-transform" strip-prefix="/api/v1"
}
```

A client's `GET /api/v1/orders/42` reaches the backend as `GET /orders/42`.

**Map an external path onto your backend's real one:**

```kdl
upstream-request {
    filter kind="uri-transform" {
        regex-uri {
            rule pattern="^/orders/(.+)$" replacement="/v2/order-service/$1"
        }
    }
}
```

**Pin an API version the backend expects, regardless of what the client sent:**

```kdl
upstream-request {
    filter kind="uri-transform" {
        set-query-param {
            param name="version" value="2"
        }
        remove-query-param {
            param name="debug"
        }
    }
}
```

## Trying it out

With this configuration:

```kdl
path-control {
    upstream-request {
        filter kind="uri-transform" strip-prefix="/api/v1"
    }
}
```

```bash
curl -v http://localhost:8080/api/v1/orders/42
```

Your backend receives `GET /orders/42` — the client's own request and the URL it used are
unaffected; only what Flow forwards changes.

## Related

- [Redirect](redirect.md) — sends the *client* elsewhere instead, never calling the backend
- [Header transform](header-transform.md) — for headers and the `Host` the backend sees
- [Policies index](../policies.md)
