# Upsert header

Sets a header, replacing any existing value. "Upsert" = update if present, insert if not.

**Phase:** `upstream-request` **or** `upstream-response`

The block you put it in decides what it acts on:

```kdl
path-control {
    // Sets a header on the request going TO the backend
    upstream-request {
        filter kind="upsert-header" key="x-forwarded-by" value="flow"
    }

    // Sets a header on the response going BACK to the client
    upstream-response {
        filter kind="upsert-header" key="x-served-by" value="flow"
    }
}
```

Note that both use the same `kind`. There is no separate `upsert-request-header` — the
enclosing block is what selects the phase.

## Options

| Option | Meaning |
|---|---|
| `key` | Header name |
| `value` | Header value |

The value is a literal string. It **replaces** any existing header of that name — an incoming
`x-forwarded-by: something-else` is overwritten, not appended to.

## Common uses

**Identify the proxy to your backend:**

```kdl
upstream-request {
    filter kind="upsert-header" key="x-forwarded-by" value="flow"
}
```

**Inject a credential the backend expects**, so clients never need to hold it:

```kdl
upstream-request {
    filter kind="upsert-header" key="x-api-key" value="backend-secret"
}
```

**Pin an API version** regardless of what the client asked for:

```kdl
upstream-request {
    filter kind="upsert-header" key="x-api-version" value="v2"
}
```

**Add a response header** for clients or CDNs:

```kdl
upstream-response {
    filter kind="upsert-header" key="cache-control" value="no-store"
}
```

## Trying it out

With this configuration:

```kdl
path-control {
    upstream-request {
        filter kind="upsert-header" key="x-api-version" value="v2"
    }
    upstream-response {
        filter kind="upsert-header" key="x-served-by" value="flow"
    }
}
```

Send a request that already carries the header:

```bash
curl -v http://localhost:8080/anything -H "x-api-version: xyz"
```

The backend sees `x-api-version: v2` — the client's `xyz` was **overwritten**, not kept. The
response you get back carries `x-served-by: flow`.

## A note on secrets

If you use this to inject a credential, remember the value sits in **plain text in your config
file**. Protect the config file's permissions accordingly, and keep it out of version control
if it holds real secrets.

## Related

- [Remove header](remove-header-key-regex.md) — to strip headers instead of setting them
- [Policies index](../policies.md)
