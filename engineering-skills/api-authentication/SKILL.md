---
name: api-authentication
description: Use this skill when choosing or implementing how API clients prove identity — API keys, OAuth 2.0, JWTs, or mutual TLS — and when wiring authorization (scopes, ownership) on top. Helps pick the right scheme for the consumer and avoid the classic auth footguns.
---

# API authentication

## When to use this

You're deciding how clients authenticate to an API, or implementing/reviewing an auth
layer. These patterns are **not strict alternatives** — most real systems layer several.
First match the scheme to the consumer, then implement carefully (auth is where the worst
security bugs live).

## Decision (match scheme to consumer)

| Scheme | Identifies | Best for | Worst for |
|---|---|---|---|
| **API key** | an account/integration | server-to-server, developer-facing APIs | end-user delegation |
| **OAuth 2.0** | a user (via an app) | end-user web/mobile apps; third-party access to user data | simple internal calls |
| **JWT** | whatever the issuer claims | microservice tokens; OAuth access-token format | sessions needing instant logout |
| **mTLS** | a machine/service | service mesh, regulated industries | browser-facing APIs |

Quick routing:
- **Developer-facing API, consumer *is* the account** (Stripe-style) → **API keys**.
- **End user signs into a web/mobile app** → **OAuth 2.0 Authorization Code + PKCE**
  (tokens are usually JWTs); add **OpenID Connect** if you need to know *who* the user is.
- **Third-party app acts on a user's behalf** → OAuth 2.0 Auth Code + PKCE.
- **Server-to-server, you own both ends** → API keys or OAuth Client Credentials.
- **High-security / regulated service-to-service** → **mTLS**, optionally JWT on top.
- **Microservices in a mesh** → mTLS for transport identity + JWT for app-level claims;
  automate cert rotation with a **service mesh**.

Common confusions to avoid: "JWT vs API keys" and "OAuth vs JWT" are false dichotomies —
JWT is a token *format*, OAuth is a *flow*, API keys are *opaque pointers*. OAuth alone
authorizes an app; it does not authenticate the user (that needs OIDC). mTLS authenticates
the *connection*, not the request — you still need app-level authorization.

## Implementation essentials

### API keys (issue → use → revoke; no login/logout)

```
sign-up → crypto.randomBytes(32) → prefix it (sk_live_<random>) →
store SHA-256 HASH + account_id + scopes → return plaintext ONCE (never retrievable again)
```

- Send in `Authorization: Bearer ...`, **never** in the URL/query string.
- Store **hashes**, not plaintext; compare with `crypto.timingSafeEqual`, never `===`.
- Attach **scopes** to the key; reject missing scope with `403` (the key is valid, just not
  authorised). Issue **scoped/restricted** keys so a leak can't drain the account.
- Revoke = delete/`revoked_at` the row → next request `401`, instant (a key advantage over JWT).
- Design prefixes (`sk_live_…`) to be secret-scanner-friendly; log per-key usage to detect compromise.

### JWT (a format — verify it correctly or it's worse than nothing)

```typescript
const { payload } = await jwtVerify(token, publicKey, {
  issuer:   'https://auth.example.com',     // opt-in: NOT checked by default
  audience: 'https://api.example.com',      // opt-in: NOT checked by default
  algorithms: ['RS256'],                    // PIN the algorithm — never trust token's `alg`
});
if (!(payload.scope as string ?? '').split(' ').includes('invoices:write'))
  return res.status(403).json({ /* forbidden problem+json */ });
```

- Use **`jose`** (panva) by default; `jsonwebtoken`/`fast-jwt` work but you **must** pass
  `algorithms`, `audience`, `issuer` explicitly or they aren't validated.
- Validate `iss`, `aud`, `exp`, `nbf` on every request.
- **Footgun checklist:** `alg:none`; RS256→HS256 algorithm confusion (public key used as HMAC
  secret); missing `aud`; weak HMAC secrets; `jku`/`jwk`/`x5u` header injection (pin the
  JWKS URL); JWE bombs. **Never hand-roll verification.**
- JWT has no clean logout (stateless). Standard pragmatic pattern: **short access tokens
  (5–15 min) + revocable refresh tokens** server-side. Browser apps: refresh token in an
  `httpOnly` cookie, access token in memory — not `localStorage`.

### OAuth 2.0 / OIDC

Use **Authorization Code + PKCE** for user-facing apps; **Client Credentials** for
machine-to-machine. Access tokens are often JWTs (verify as above). Add **OpenID Connect**
(ID token) when you need authenticated identity, not just delegated authorization.

### mTLS

Both sides present X.509 certs; identity lives in the TLS handshake (no header). Right for
service-to-service in regulated environments and meshes. High cert-management cost — a
**service mesh** automates issuance, rotation, and policy across many services.

## Authorization (every scheme)

Authentication ≠ authorization. Claims/scopes tell you *what kind* of access; they do **not**
prove *this user owns this resource*. Always add **object-level ownership checks** (the BOLA
gap) for `/orders/:id`-style routes. Remember JWT claims can be **stale** — for changes that
must take effect immediately (role downgrade, ban), do a server-side check or use very short
tokens.

## Checklist

- [ ] Scheme matches the consumer (account vs end user vs machine).
- [ ] Credentials in headers, never in URLs/query strings.
- [ ] API keys: hashed at rest, `timingSafeEqual`, scoped, revocable, usage-logged.
- [ ] JWT: algorithm pinned; `iss`/`aud`/`exp`/`nbf` validated; vetted library; no `localStorage`.
- [ ] Short access tokens + revocable refresh tokens; a real logout story.
- [ ] `401` for unauthenticated, `403` for unauthorised; generic "Invalid credentials" (no enumeration).
- [ ] Object-level ownership checks on resource routes (BOLA).

## Pitfalls

- **Keys in URLs** (logs, history, referrers); **storing plaintext keys**; **non-constant-time compare**.
- **Single all-powerful key**; **no rotation strategy**; **no usage telemetry**.
- **Hand-rolling JWT verification**; **long token expiry**; **treating JWT as a session**.
- **Missing `aud`/`alg` pinning** — token for service A works on B / `alg:none` forgery.
- **Assuming OAuth authenticates the user** (it doesn't — add OIDC).
- **Treating mTLS as full authz** — it authenticates the connection only.
- **Skipping object-level ownership checks** — valid token, someone else's data.

## Reference

Vault notes:
- `API Best Practices/Authentication & Authorization/Authentication Strategies Compared.md`
- `API Best Practices/Authentication & Authorization/API Keys.md`
- `API Best Practices/Authentication & Authorization/OAuth 2.0.md`
- `API Best Practices/Authentication & Authorization/JSON Web Tokens.md`
- `API Best Practices/Authentication & Authorization/Mutual TLS.md`
- `API Best Practices/Authentication & Authorization/Service Mesh.md`

External: OWASP API Security Top 10 (https://owasp.org/API-Security/) ·
OAuth 2.0 Simplified (https://www.oauth.com/) ·
RFC 7519 JWT (https://datatracker.ietf.org/doc/html/rfc7519) · jwt.io (https://jwt.io/)

Related skills: `api-rate-limiting` (key limits on the identity this establishes),
`rest-api-error-design` (401/403 bodies, no enumeration), `express-production-api` (where auth sits).
