# Header transform

Removes and/or sets headers in one filter. "Upsert" = update if present, insert if not.

**Phase:** `upstream-request` **or** `upstream-response`

The block you put it in decides what it acts on:

```kdl
path-control {
    // Acts on the request going TO the backend
    upstream-request {
        filter kind="header-transform" {
            remove {
                header pattern=".*(secret|SECRET).*"
            }
            upsert {
                header name="x-forwarded-by" value="flow"
            }
        }
    }

    // Acts on the response going BACK to the client
    upstream-response {
        filter kind="header-transform" {
            upsert {
                header name="x-served-by" value="flow"
            }
        }
    }
}
```

Both use the same `kind` — the enclosing block selects the phase.

## Shape

One `header-transform` filter can carry a `remove` block, an `upsert` block, or both — each
holding one or more `header` rows:

```kdl
filter kind="header-transform" {
    remove {
        header pattern="^x-internal-.*"
        header pattern="(?i)^(server|x-powered-by)$"
    }
    upsert {
        header name="x-api-version" value="v2"
        header name="x-forwarded-by" value="flow"
    }
}
```

**Execution order: `remove`, then `upsert`.** If a `remove` pattern and an `upsert` row target
the same header name, the removal runs first, so the upserted value survives.

## Options

| Block | Row | Meaning |
|---|---|---|
| `remove` | `header pattern="..."` | Regex matched against header **names**; every matching header is removed |
| `upsert` | `header name="..." value="..."` | Sets a header, replacing any existing value |

`upsert`'s value is a literal string. It **replaces** any existing header of that name — an
incoming `x-forwarded-by: something-else` is overwritten, not appended to.

`remove`'s pattern matches header **names**, not values — you cannot use this to remove a
header based on what it contains.

## Case sensitivity

`remove`'s pattern match is **always case-insensitive**. HTTP header names are
case-insensitive by protocol (RFC 7230 §3.2), so `pattern=".*etag.*"` matches `ETag`,
`Etag`, and `etag` alike — no `(?i)` flag needed.

## Anchor your patterns

`.*secret.*` also matches `x-not-a-secret-at-all`. If you mean an exact header name, anchor
it:

```kdl
remove {
    header pattern="^x-secret-key$"
}
```

An over-broad pattern silently strips headers you meant to keep, which can be hard to
diagnose — the header simply isn't there.

## Common uses

**Identify the proxy to your backend, and strip internal headers before they reach it:**

```kdl
upstream-request {
    filter kind="header-transform" {
        remove {
            header pattern="^x-internal-.*"
        }
        upsert {
            header name="x-forwarded-by" value="flow"
        }
    }
}
```

**Inject a credential the backend expects**, so clients never need to hold it:

```kdl
upstream-request {
    filter kind="header-transform" {
        upsert {
            header name="x-api-key" value="backend-secret"
        }
    }
}
```

**Pin an API version** regardless of what the client asked for:

```kdl
upstream-request {
    filter kind="header-transform" {
        upsert {
            header name="x-api-version" value="v2"
        }
    }
}
```

**Scrub backend implementation details and add a response header** for clients or CDNs:

```kdl
upstream-response {
    filter kind="header-transform" {
        remove {
            header pattern="^(server|x-powered-by)$"
        }
        upsert {
            header name="cache-control" value="no-store"
        }
    }
}
```

## Trying it out

With this configuration:

```kdl
path-control {
    upstream-request {
        filter kind="header-transform" {
            remove {
                header pattern=".*(secret|SECRET).*"
            }
            upsert {
                header name="x-api-version" value="v2"
            }
        }
    }
    upstream-response {
        filter kind="header-transform" {
            upsert {
                header name="x-served-by" value="flow"
            }
        }
    }
}
```

Send a request that carries both headers:

```bash
curl -v http://localhost:8080/anything -H "x-api-version: xyz" -H "Secret: abc"
```

The backend sees `x-api-version: v2` — the client's `xyz` was **overwritten**, not kept — and
no `Secret` header at all. The response you get back carries `x-served-by: flow`.

## A note on secrets

If you use `upsert` to inject a credential, remember the value sits in **plain text in your
config file**. Protect the config file's permissions accordingly, and keep it out of version
control if it holds real secrets.

## Related

- [URI transform](uri-transform.md) — for the path and query string the backend sees, not headers
- [Policies index](../policies.md)
