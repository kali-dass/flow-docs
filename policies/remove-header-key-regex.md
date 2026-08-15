# Remove header

Removes every header whose **name** matches a regular expression.

**Phase:** `upstream-request` **or** `upstream-response`

The block you put it in decides what it acts on:

```kdl
path-control {
    // Strips headers from the request going TO the backend
    upstream-request {
        filter kind="remove-header-key-regex" pattern=".*(secret|SECRET).*"
    }

    // Strips headers from the response going BACK to the client
    upstream-response {
        filter kind="remove-header-key-regex" pattern=".*[Ee][Tt][Aa][Gg].*"
    }
}
```

Both use the same `kind` — the enclosing block selects the phase.

## Options

| Option | Meaning |
|---|---|
| `pattern` | Regex matched against header **names** |

> The pattern matches header **names**, not values. You cannot use this to remove a header
> based on what it contains.

## Common uses

**Strip internal headers before they reach a backend** — so a client can't spoof them:

```kdl
upstream-request {
    filter kind="remove-header-key-regex" pattern="^x-internal-.*"
}
```

**Scrub backend implementation details out of responses:**

```kdl
upstream-response {
    filter kind="remove-header-key-regex" pattern="(?i)^(server|x-powered-by)$"
}
```

**Remove caching headers** you want to control yourself:

```kdl
upstream-response {
    filter kind="remove-header-key-regex" pattern="(?i).*etag.*"
}
```

## Case sensitivity

The pattern is matched as written, so it is **case-sensitive by default**. HTTP header names
are case-insensitive in practice, so make your pattern case-insensitive too — either with the
inline `(?i)` flag:

```kdl
pattern="(?i)^authorization$"
```

or by spelling out the alternatives:

```kdl
pattern=".*(secret|SECRET).*"
```

**Prefer `(?i)`** — it's harder to get wrong than enumerating cases.

## Anchor your patterns

`.*secret.*` also matches `x-not-a-secret-at-all`. If you mean an exact header name, anchor
it:

```kdl
pattern="(?i)^x-secret-key$"
```

An over-broad pattern silently strips headers you meant to keep, which can be hard to
diagnose — the header simply isn't there.

## Trying it out

With `pattern=".*(secret|SECRET).*"` on `upstream-request`:

```bash
curl -v http://localhost:8080/anything -H "Secret: abc" -H "Accept: application/json"
```

The backend receives the request **without** the `Secret` header. `Accept` passes through
untouched.

## Related

- [Upsert header](upsert-header.md) — to set headers instead of stripping them
- [Policies index](../policies.md)
