---
tags:
  - api-design
  - authentication
  - authorization
  - security
  - oauth
created: 2026-06-03
---

# OAuth 2.0

> Summary: A framework (RFC 6749) for *delegated authorization* — letting an app act on a user's behalf without their password. Defines several flows; the only ones to use today are **Authorization Code + PKCE** (user-facing) and **Client Credentials** (server-to-server). Implicit and ROPC are deprecated. OAuth alone doesn't tell you *who* the user is — that's what [[OpenID Connect]] adds on top. Access tokens are often (but not always) [[JSON Web Tokens|JWTs]].

## The flows that matter

| Flow | Use case |
|---|---|
| **Authorization Code + PKCE** | Web/mobile apps acting on behalf of a user. "Sign in with Google." Default. |
| **Client Credentials** | Server-to-server, no user. Often better than long-lived [[API Keys]]. |
| **Device Authorization** (RFC 8628) | CLIs, smart TVs, IoT — no good input method. |
| ~~Implicit~~, ~~Resource Owner Password Credentials~~ | Deprecated. Don't use. |

> [!tip] PKCE is mandatory
> As of OAuth 2.1, PKCE (RFC 7636) is required for *all* clients — public and confidential. If a tutorial says PKCE is only for mobile/SPA, it's out of date.

## Flow (Authorization Code + PKCE)

```
┌─────────┐    ┌──────────┐    ┌──────────────┐    ┌─────────┐
│ Browser │    │ Your App │    │ Auth Server  │    │ Your API│
└────┬────┘    └────┬─────┘    └──────┬───────┘    └────┬────┘
     │ 1. click "Login with X"        │                  │
     │ ──────────────►│               │                  │
     │                │ 2. generate state + PKCE pair    │
     │                │    redirect with code_challenge  │
     │ ◄──────────────│               │                  │
     │                                │                  │
     │ 3. GET /authorize?client_id=...&redirect_uri=...  │
     │    &state=XXX&code_challenge=YYY&scope=openid+... │
     │ ──────────────────────────────►│                  │
     │                                │                  │
     │ 4. login form + consent screen │                  │
     │ ◄──────────────────────────────│                  │
     │ 5. submit credentials          │                  │
     │ ──────────────────────────────►│                  │
     │                                │                  │
     │ 6. redirect with ?code=ZZZ&state=XXX              │
     │ ◄──────────────────────────────│                  │
     │                                │                  │
     │ 7. GET /callback?code=ZZZ&state=XXX               │
     │ ──────────────►│               │                  │
     │                │ 8. verify state matches          │
     │                │                                  │
     │                │ 9. POST /token                   │
     │                │    { code, code_verifier,        │
     │                │      client_id, client_secret }  │
     │                │ ────────────────►                │
     │                │                                  │
     │                │ 10. { access_token, refresh_token, expires_in, id_token }
     │                │ ◄────────────────                │
     │                │                                  │
     │ 11. set httpOnly session cookie, redirect home    │
     │ ◄──────────────│               │                  │
     │                │                                  │
     │ ── later, calling the API ──                      │
     │                │                                  │
     │ 12. user action │                                 │
     │ ──────────────►│                                  │
     │                │ 13. Authorization: Bearer <access_token>
     │                │ ──────────────────────────────────►│
     │                │                                    │ 14. verify signature,
     │                │                                    │     exp, aud, iss,
     │                │                                    │     check scope
     │                │ 15. data                           │
     │                │ ◄──────────────────────────────────│
     │ 16. response   │                                  │
     │ ◄──────────────│                                  │
```

Key things to notice:

- The user's password never touches *your app* — only the auth server sees it.
- The `code` is one-time-use and short-lived. It's exchanged for the actual token over a backchannel (server-to-server), with `code_verifier` proving the original PKCE challenge.
- Your app stores tokens; the browser holds a session cookie, not the token itself (safer against XSS).
- `id_token` is only returned if you requested `scope=openid` — that's the [[OpenID Connect]] bit.

## Register / login / logout

### Register

In pure OAuth, **you don't register users** — the auth server handles user accounts. Two common cases:

- **Federated identity** (Sign in with Google/GitHub/Apple): user already has an account at the provider; your app gets identity claims via [[OpenID Connect]]. There's no "register" step in your app — first login auto-creates a local record.
- **Your own auth server** (Auth0, Keycloak, Ory, Clerk, Supabase Auth, etc.): the auth server has a registration UI that creates accounts. Your app delegates the entire signup flow.

### Login

The full Authorization Code + PKCE dance above. From the user's perspective: click "Login," redirect to a login screen, get redirected back, logged in. Your app never sees credentials.

### Logout

Two layers — both matter:

1. **Local logout (your app):** clear the session cookie, drop stored tokens, optionally call the auth server's `/revoke` endpoint to kill the refresh token server-side.
2. **Global logout (the auth server):** call the OIDC `end_session_endpoint`. Without this, the user is logged out of *your* app but still has an active session at the IdP — clicking "Login" again would silently sign them right back in.

If you only do local logout, "logout" isn't really logout — they just have to click once more.

## Authorization (what the user can access)

Scopes are requested at login and granted by the user during consent:

```http
GET /authorize?...&scope=openid+profile+invoices:read+invoices:write
```

After consent, the access token (often a JWT) carries the granted scopes:

```json
{ "sub": "user_42", "scope": "invoices:read invoices:write", ... }
```

On API requests:

```
Route POST /invoices requires scope "invoices:write"
Token has "invoices:read invoices:write" → allowed
Token has only "invoices:read" → 403 Forbidden
```

> [!warning] Scopes are coarse-grained
> Scopes answer "can this app do this *kind* of thing?" They don't answer "does user 42 own *this specific* invoice?" For object-level authorization, you still need application-level checks. **OWASP API #1 (BOLA) breaks at exactly this gap.**

## Common mistakes

> [!warning] Anti-patterns
> - **Missing `state` parameter** — opens CSRF on the auth code flow.
> - **Open redirects via `redirect_uri`.** Strict exact-match allowlist. No wildcards, no prefixes.
> - **Skipping PKCE.** Mandatory in OAuth 2.1.
> - **No refresh token rotation.** A leaked refresh token lives forever.
> - **Only local logout.** Users stay signed in at the IdP.
> - **Building your own auth server.** Use Keycloak, Ory Hydra, Auth0, Okta, Clerk, Supabase.
> - **Confusing OAuth with authentication.** OAuth = "this app can do X on the user's behalf." Identity = [[OpenID Connect]] on top.
> - **Relying only on scopes for authorization.** Always add object-level checks.

## Resources for TypeScript implementation

**`openid-client`** (by panva) — the recommended default
- *Why use:* OpenID Certified™. Covers OAuth 2.0, 2.1, OIDC, FAPI, DPoP, CIBA.
- *Why not:* overkill for the simplest single-provider case.
- *Handles automatically:* PKCE, state, token exchange, refresh, JWKS fetching with rotation, claim validation.
- *Be aware of:* you configure the `redirect_uri` allowlist server-side, store tokens securely (httpOnly cookies, not localStorage), and decide refresh-token rotation policy.

**`oauth4webapi`** (also by panva) — low-level
- *Why use:* smaller bundle, runs in edge runtimes (Cloudflare Workers, Deno, Vercel Edge).
- *Why not:* more code to write yourself.
- *Handles automatically:* crypto operations, spec-compliant request shapes.
- *Be aware of:* you wire the flow steps yourself; easy to skip a check.

**`Auth.js`** (formerly NextAuth.js)
- *Why use:* drop-in for Next.js, SvelteKit, Express, Hono. Many providers preconfigured. Handles register, login, logout, session out of the box.
- *Why not:* opinionated session model; harder if you need non-standard claims or flows.
- *Handles automatically:* callbacks, sessions, CSRF on auth routes, provider quirks, JWT signing.
- *Be aware of:* database adapter choice matters for refresh tokens; custom claims require callback hooks; check what's in the JWT vs. database session.

**`Passport` + strategies** (`passport-oauth2`, `passport-google-oauth20`, ...)
- *Why use:* huge ecosystem, Express-native, familiar.
- *Why not:* older callback-heavy patterns; strategy quality varies; some are unmaintained.
- *Handles automatically:* per-strategy provider quirks.
- *Be aware of:* serialization/deserialization, session middleware setup, and that "it works" ≠ "it's secure" — audit the strategy.

## References

- RFC 6749 (OAuth 2.0): https://datatracker.ietf.org/doc/html/rfc6749
- RFC 7636 (PKCE): https://datatracker.ietf.org/doc/html/rfc7636
- OAuth 2.0 Security BCP: https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/
- OAuth 2.1 draft: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Aaron Parecki, *OAuth 2.0 Simplified*: https://www.oauth.com/

## Related

- [[API Keys]]
- [[JSON Web Tokens]]
- [[OpenID Connect]]
- [[Mutual TLS]]
- [[Authentication strategies compared]]
- [[Authentication and Authorization patterns]]