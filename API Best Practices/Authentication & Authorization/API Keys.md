---
tags:
  - api-design
  - authentication
  - security
  - api-keys
created: 2026-06-03
---

# API Keys

> Summary: A long random string sent with every request, identifying an *account or integration* (not usually a human end-user). The simplest auth scheme — opaque to the client, looked up server-side on every call. No session, no login/logout — keys are long-lived credentials. Right for server-to-server traffic and developer-facing APIs where the consumer *is* the account. Wrong for end-user-facing apps — use [[OAuth 2.0]] instead.

## Wire format

Always in a header — never a query string.

```http
GET /v1/charges HTTP/1.1
Host: api.example.com
Authorization: Bearer sk_live_abc123XYZ...
```

Some APIs use a custom header like `X-API-Key`. Functionally equivalent; `Authorization: Bearer` is more conventional.

## Flow

```
┌─────────┐                                       ┌─────────┐
│ Client  │                                       │ Server  │
└────┬────┘                                       └────┬────┘
     │  GET /api/users                                 │
     │  Authorization: Bearer sk_live_abc...           │
     │ ──────────────────────────────────────────────► │
     │                                                 │ 1. Extract key 
     |                                                 |    from header
     │                                                 │ 2. Hash key, 
     |                                                 |    look up in DB
     │                                                 │ 3. Check scopes 
     |                                                 |    vs. route
     │                                                 │ 4. Log usage
     │  200 OK + JSON                                  │
     │ ◄────────────────────────────────────────────── │
```

Every request is independent. No session, no token to refresh, no state on the client beyond "store this key somewhere safe."

## Lifecycle (there is no login/logout)

API keys don't have a login or logout flow because there's no session — they're long-lived credentials. The lifecycle is **issue → use → revoke**.

### Issue (the "register" equivalent)

```
User signs up
      ↓
Server generates 32 random bytes (crypto.randomBytes)
      ↓
Prepend a recognisable prefix → sk_live_<random>
      ↓
Hash the key (SHA-256) and store hash + account_id + scopes in DB
      ↓
Return the plaintext key to the user ONCE
      ↓
User stores it (env var, secret manager); server cannot retrieve it again
```

The plaintext key never gets persisted server-side. If the user loses it, they generate a new one — same security model as a password reset.

### Use (every request)

Client sends the key in `Authorization: Bearer`. Server hashes the incoming key the same way and looks up the hash. **Use `crypto.timingSafeEqual` for comparison, not `===`.**

### Revoke (the "logout" equivalent)

To revoke: delete (or mark `revoked_at`) the row. The next request with that key returns `401 Unauthorized`. **Instant** — no token expiry to wait for. This is one of API keys' biggest advantages over [[JSON Web Tokens|JWTs]].

Every key should be revocable independently, with a record of who issued it, when, and what it's used for.

## Authorization (what the key can access)

Scopes/permissions live alongside the key in the database:

```sql
CREATE TABLE api_keys (
  id            UUID PRIMARY KEY,
  key_hash      TEXT NOT NULL,
  account_id    UUID NOT NULL,
  scopes        TEXT[] NOT NULL,  -- e.g., ['invoices:read', 'invoices:write']
  created_at    TIMESTAMPTZ NOT NULL,
  last_used_at  TIMESTAMPTZ,
  revoked_at    TIMESTAMPTZ
);
```

On every request:

1. Hash the incoming key, look up the row.
2. Check the route's required scope against `scopes`.
3. Reject with **`403 Forbidden`** if missing — not `401`. The key is valid; it's just not authorised for this action.

Stripe's "restricted API keys" are the canonical example: generate a key scoped to a specific use (CI deploys, a read-only dashboard) so a leak can't drain the account.

## Common mistakes

> [!warning] Anti-patterns
> - **Keys in URLs** (`?api_key=...`) — ends up in server logs, browser history, referrer headers.
> - **Storing plaintext keys.** Hash them like passwords; you only need to *verify*, not retrieve.
> - **Non-constant-time comparison.** Use `crypto.timingSafeEqual`, not `===`.
> - **Single all-powerful key.** A leak is catastrophic. Issue scoped keys.
> - **No rotation strategy.** You'll need it under incident pressure — build it in from day one.
> - **No usage telemetry.** If you can't see "this key just started calling from a new country," you can't detect compromise.
> - **Hardcoded in repos.** Design key prefixes (`sk_live_...`) to be scanner-friendly.

## Resources for TypeScript implementation

API keys are simple enough that most teams roll their own middleware. The library landscape is thin.

**Custom Express/Fastify/Hono middleware** (the typical choice)
- *Why use:* the pattern is ~20 lines of code; no library buys you much.
- *Why not:* you must remember the security details yourself (constant-time compare, hashing at rest, scopes).
- *Handles automatically:* nothing — you write it.
- *Be aware of:* hash keys at rest, constant-time compare, generate with `crypto.randomBytes(32)`, attach scopes to the key in the DB, log usage per key.

**`passport-headerapikey`** (Passport strategy)
- *Why use:* if you're already using Passport for other auth.
- *Why not:* drags in Passport's session machinery for what should be a one-liner.
- *Handles automatically:* header extraction, strategy plumbing.
- *Be aware of:* you still write the key-validation logic; security choices are yours.

**Framework built-ins** (NestJS `@UseGuards()`, Hono auth helpers, etc.)
- *Why use:* idiomatic in those frameworks.
- *Why not:* lock-in.
- *Handles automatically:* request interception, DI.
- *Be aware of:* you still implement verification inside the guard.

## References

- OWASP API Security Top 10: https://owasp.org/API-Security/
- Stripe's API key format / scanning: https://stripe.com/blog/canonical-log-lines

## Related

- [[OAuth 2.0]]
- [[JSON Web Tokens]]
- [[Mutual TLS]]
- [[Authentication strategies compared]]
- [[Authentication and Authorization patterns]]
- [[Rate limiting]]