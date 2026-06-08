---
tags:
  - api-design
  - authentication
  - security
  - jwt
  - tokens
created: 2026-06-03
---

# JSON Web Tokens

> Summary: A **token format** (RFC 7519), not an auth scheme. Three Base64URL-encoded JSON blobs joined by dots: header, payload (claims), signature. Lets services verify authenticity statelessly using the issuer's public key — useful for microservices. Often the format of access tokens issued by [[OAuth 2.0]], but also used standalone for homegrown auth. Has a long history of CVEs caused by misuse — never hand-roll verification.

## Anatomy

```
eyJhbGciOiJSUzI1NiIs...   .   eyJzdWIiOiI0MiIsImV4cCI6MTcz...   .   <signature>
   ↑ header                       ↑ payload (claims)                ↑ signature
```

## Standard claims (validate on every request)

| Claim | Meaning | Why validate |
|---|---|---|
| `iss` | Issuer | Token came from a trusted auth server |
| `sub` | Subject | Who the token is about |
| `aud` | Audience | Token was issued *for your API*, not someone else's |
| `exp` | Expiration | Token isn't expired |
| `iat` | Issued at | Not from the future (clock skew) |
| `nbf` | Not before | Not being used too early |
| `jti` | JWT ID | Unique identifier — useful for revocation lists |

Most libraries auto-check `exp`. You typically have to **opt in** to `aud` and `iss` checks. **Do it.**

## Flow (homegrown JWT auth)

JWT is a *format* — it can be used inside many auth schemes. The most common standalone pattern: issue JWTs after a username/password login.

```
┌─────────┐                                       ┌─────────┐
│ Client  │                                       │ Server  │
└────┬────┘                                       └────┬────┘
     │  POST /login                                    │
     │  { "email": "...", "password": "..." }          │
     │ ──────────────────────────────────────────────► │
     │                                                 │ 1. Look up user
     │                                                 │ 2. Verify password
     |                                                 |    (Argon2id)
     │                                                 │ 3. Sign JWT:
     │                                                 │    { sub, exp, 
     |                                                 |      scope, role }
     │  200 OK                                         │
     │  { "access_token": "eyJ...",                    │
     │    "refresh_token": "..." }                     │
     │ ◄────────────────────────────────────────────── │
     │                                                 │
     │ ── client stores tokens ──                      │
     │                                                 │
     │  GET /api/me                                    │
     │  Authorization: Bearer eyJ...                   │
     │ ──────────────────────────────────────────────► │
     │                                                 │ 4. Verify signature
     │                                                 │ 5. Validate exp, 
     |                                                 |    aud, iss
     │                                                 │ 6. Check scope/role 
     |                                                 |    vs route
     │  200 OK + JSON                                  │
     │ ◄────────────────────────────────────────────── │
```

The server does no DB lookup during step 4–6 — that's the stateless promise. Identity and permissions come from the token's claims, verified by the signature alone.

## Register / login / logout

### Register

```
POST /register
{ "email": "...", "password": "..." }
     ↓
1. Validate email format, password strength
2. Hash password with Argon2id (or bcrypt as fallback)
3. INSERT into users table
4. Optionally: sign JWT and return immediately, or require separate /login
```

### Login

`POST /login` with credentials. Server verifies the password against the stored hash (constant-time), then signs a JWT. Typical pattern:

- **Access token** — short-lived JWT (5–15 min), sent on every request.
- **Refresh token** — long-lived opaque string (or rotating JWT), stored server-side. Used to mint new access tokens without re-entering credentials.

### Logout

Here's the catch: **JWT was designed for stateless verification, but logout is inherently stateful.**

Three options:

1. **Client-side only.** Delete the token on the client. The token remains cryptographically valid until `exp`, but nobody has it. Simple, but a stolen token still works.
2. **Token blocklist.** Server keeps a list of `jti`s to reject before their `exp`. Defeats statelessness but enables instant logout.
3. **Short access tokens + refresh token revocation.** Access tokens are so short-lived (~5 min) that you accept that window of validity post-logout, and revoke the refresh token server-side so no new ones can be minted.

Most production systems use option 3 — pragmatic balance.

### Where to store the token (client side)

- **`localStorage`** — easy but reachable by any JS on the page. XSS = token stolen.
- **`httpOnly` cookies** — not reachable by JS. Pair with `SameSite=Lax` (or `Strict`) and CSRF protection. Safer for browser apps.
- **Native app secure storage** (iOS Keychain, Android Keystore) — right answer for mobile.

For browsers: default to `httpOnly` cookies for refresh tokens; access tokens can sit in memory.

## Authorization (what the user can access)

Claims in the token drive authorization:

```json
{
  "sub": "user_42",
  "scope": "invoices:read invoices:write",
  "role": "admin",
  "exp": 1735693200
}
```

The server validates the token and checks claims on every request:

```typescript
const { payload } = await jwtVerify(token, publicKey, {
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
  algorithms: ['RS256']
});

const scopes = (payload.scope as string)?.split(' ') ?? [];
if (!scopes.includes('invoices:write')) {
  return reply.status(403).send({ type: '...', title: 'Forbidden' });
}
```

> [!warning] Claims can be stale
> Claims are baked in when the token is issued. If you change a user's role at 10am, their token from 9:55am still says the old role until it expires. For role changes that must take effect immediately, use short tokens + refresh, or fall back to a server-side check for sensitive operations.

> [!warning] Same BOLA gap as OAuth
> Claims tell you *what kind* of access — they don't tell you whether *this specific user* owns *this specific resource*. Always add object-level ownership checks. See [[OAuth 2.0]] for the same pitfall.

## Common mistakes (beyond the CVEs)

> [!warning] Anti-patterns
> - **Treating JWT as a session.** JWTs aren't easily revocable before `exp`. If you need instant logout, use opaque tokens with server-side state.
> - **Long expiry windows.** Standard pattern: short access tokens (5–15 min) + refresh tokens.
> - **Stuffing too much in claims.** Tokens grow; every change requires reissue.
> - **Hand-rolling verification.** Use a vetted library. Always.
> - **Storing JWTs in `localStorage`** for browser apps — XSS exposure.

## The footguns (vulnerabilities)

> [!warning] JWT vulnerability checklist
> - **`alg: none` attack.** Always pin the algorithm server-side. Never trust the token's claimed `alg`.
> - **Algorithm confusion (RS256 → HS256).** Attacker submits `alg: HS256` signed with your *public key* as the HMAC secret. Pin algorithm + use distinct key objects. (CVE-2024-33663 in python-jose, CVE-2022-29217 in PyJWT.)
> - **Missing `aud` validation.** Token meant for service A works on service B.
> - **Missing `exp` / `nbf` checks.** Stale tokens still work.
> - **Weak HMAC secrets.** `HS256` with "secret123" is brute-forceable offline. Use ≥256-bit random secrets, or prefer RS256/ES256.
> - **`jku` / `jwk` / `x5u` header injection.** Verifier fetches the key from a URL in the token → attacker controls verification. Pin the JWKS URL server-side.
> - **JWT bomb DoS.** Compressed JWE with huge expansion ratio. Limit JWE size; don't accept JWE unless needed. (CVE-2024-33664 in python-jose, CVE-2024-21319 in Microsoft.)

## Resources for TypeScript implementation

**`jose`** (by panva) — recommended default
- *Why use:* TypeScript-native, runs in Node/browser/Cloudflare Workers/Deno/Bun. Supports JWS, JWE, JWK, JWKS. Default in Auth.js and Hono.
- *Why not:* slightly more verbose API than `jsonwebtoken`.
- *Handles automatically:* algorithm enforcement (you pass an `algorithms` array), `exp`/`nbf` checks, JWKS fetching with rotation and caching.
- *Be aware of:* you must explicitly pass expected `audience` and `issuer` — they're not checked by default. Pin algorithms; never pass an empty array.

**`jsonwebtoken`**
- *Why use:* huge installed base (~20M weekly downloads), simpler synchronous-feeling API.
- *Why not:* legacy; no JWE/JWKS support; slow to ship modern features. Reach for `jose` in new code.
- *Handles automatically:* signing, basic `exp` check.
- *Be aware of:* you **must** pass an `algorithms` array to `verify()` — otherwise the library trusts the token's claimed `alg`. Same for `audience` and `issuer` — opt in or they aren't checked.

**`fast-jwt`**
- *Why use:* performance-focused; faster sign/verify than alternatives.
- *Why not:* smaller ecosystem; fewer tutorials.
- *Handles automatically:* same baseline as jsonwebtoken.
- *Be aware of:* same algorithm-pinning and claim-validation requirements as any JWT library.

> [!tip] Don't issue JWTs unless you need to
> If you don't have multiple services that need stateless verification, opaque tokens with a server-side store are simpler, easier to revoke, and have fewer footguns.

## References

- RFC 7519 (JWT): https://datatracker.ietf.org/doc/html/rfc7519
- jwt.io decoder + library list: https://jwt.io/
- Auth0 JWT Handbook (free PDF): https://auth0.com/resources/ebooks/jwt-handbook
- CVE-2024-33663 (algorithm confusion): https://www.cve.org/CVERecord?id=CVE-2024-33663

## Related

- [[OAuth 2.0]]
- [[API Keys]]
- [[Mutual TLS]]
- [[Authentication strategies compared]]
- [[Authentication and Authorization patterns]]
- [[OpenID Connect]]