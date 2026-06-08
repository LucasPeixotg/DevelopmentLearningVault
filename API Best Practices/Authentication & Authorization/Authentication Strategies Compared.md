---
tags: [api-design, authentication, authorization, security]
created: 2026-06-03
---

# Authentication strategies compared

> Summary: A side-by-side of the four main API auth patterns — [[API Keys]], [[OAuth 2.0]], [[JSON Web Tokens]], and [[Mutual TLS]]. The TL;DR: API keys identify integrations, OAuth delegates user authority to apps, JWT is a token *format* (often used by OAuth), and mTLS authenticates connections at the transport layer. They aren't strict alternatives — most real systems use several together.

## What each one is

- **API key** — an opaque random string identifying an *account or integration*. Looked up server-side on every call.
- **OAuth 2.0** — a *framework* for letting an app act on a user's behalf without their password. Issues access tokens (often [[JSON Web Tokens|JWTs]]).
- **JWT** — a signed token *format*. Self-describing claims verified cryptographically. Stateless.
- **mTLS** — both sides of a TLS connection present X.509 certs. Identity lives in the transport layer, not HTTP.

## Side-by-side

| | API Key | OAuth 2.0 | JWT | mTLS |
|---|---|---|---|---|
| **Type** | Auth scheme | Authorization framework | Token format | Auth scheme |
| **Identifies** | An account/integration | A user (via app) | Whatever the issuer says | A machine/service |
| **Lives where on the wire** | HTTP header | HTTP header (Bearer) | HTTP header (Bearer) | TLS handshake (no header) |
| **Opaque or structured** | Opaque | Either (often JWT) | Structured | Cert (structured) |
| **Verification** | DB lookup | DB lookup or signature | Signature only | TLS handshake + CA |
| **Stateful or stateless** | Stateful | Either | Stateless | Stateless (per connection) |
| **Typical lifetime** | Months to years | Minutes to hours (refresh tokens longer) | Minutes to hours | Months to years (cert validity) |
| **Revocation** | Easy (delete row) | Easy for refresh, hard for access | Hard before `exp` | Hard (CRL/OCSP) |
| **Carries permissions** | Looked up server-side | In claims or looked up | In claims | In cert extensions, or mapped server-side |
| **Setup cost** | Trivial | Moderate to high | Low (with library) | High (cert infra) |
| **Best for** | Server-to-server, dev-facing APIs | End-user-facing apps | Microservice tokens, OAuth access tokens | Service mesh, regulated industries |
| **Worst for** | End-user delegation | Simple internal calls | Sessions needing instant logout | Browser-facing APIs |

## Decision guide

> [!tip] What to reach for
> - **Developer-facing API where the consumer is an account** (Stripe-style) → [[API Keys]]
> - **End-user signs into a web/mobile app** → [[OAuth 2.0]] Authorization Code + PKCE, tokens are usually [[JSON Web Tokens|JWTs]]
> - **Third-party app accesses your users' data on their behalf** → [[OAuth 2.0]] Authorization Code + PKCE
> - **Server-to-server, you control both ends, internal** → [[API Keys]] or OAuth Client Credentials
> - **Server-to-server, high security or regulated** → [[Mutual TLS]], optionally with JWT on top
> - **Microservices in a mesh** → [[Mutual TLS]] for transport, JWT for app-level claims
> - **CLI or smart-TV login** → OAuth Device Authorization flow

## Common combinations

These patterns layer naturally. Most production systems use several:

- **OAuth 2.0 issues JWTs.** OAuth is the *flow*, JWT is the *format* of the token it issues. "We use JWT auth" usually means "we use OAuth that issues JWTs."
- **API key + JWT.** Developer's API key identifies the *integration*; a JWT inside that call identifies the *end user* on whose behalf the integration is acting.
- **mTLS + JWT.** Service mesh handles machine identity via mTLS; the JWT inside the request carries the end-user identity.
- **API key + mTLS.** Partner APIs in regulated industries — the cert proves which partner, the API key adds an application-level credential.

## Common confusions

> [!warning] These trip people up
> - **"JWT vs API keys"** is a false dichotomy. JWTs and API keys both ride in `Authorization: Bearer`, but JWTs are *self-describing* (claims inside, verified by signature), while API keys are *opaque pointers* (meaning lives server-side). Stripe's `sk_live_...` is an API key. A Google access token is a JWT.
> - **"OAuth vs JWT"** is also a false dichotomy. OAuth is a *flow* for getting tokens; JWT is one *format* those tokens can take. You can have OAuth without JWTs (opaque tokens), and JWTs without OAuth (signed by your own service).
> - **OAuth alone doesn't authenticate the user.** It authorizes an app to act on a user's behalf. Knowing *who* the user is requires [[OpenID Connect]] on top.
> - **mTLS authenticates the connection, not the request.** Once the TLS connection is established, every request on it is "authenticated" — you still need application-level authorization (which user, which scope).

## References

- OWASP API Security Top 10: https://owasp.org/API-Security/
- Aaron Parecki, *OAuth 2.0 Simplified*: https://www.oauth.com/

## Related

- [[API Keys]]
- [[OAuth 2.0]]
- [[JSON Web Tokens]]
- [[Mutual TLS]]
- [[OpenID Connect]]
- [[Authentication and Authorization patterns]]