---
tags: [api-design, authentication, security, mtls, tls, best-practices]
created: 2026-06-03
---
# Mutual TLS

> Summary: Both sides of a TLS connection present X.509 certificates; the server verifies the client's cert as well as the other way around. Identity lives in the transport layer, not HTTP — no `Authorization` header is needed. Strong cryptographic identity with no bearer tokens to leak, but operationally heavy. Standard in service meshes, regulated finance (UK/EU Open Banking mandates it), and zero-trust networking. Login/logout don't apply — identity is per *connection*, not per session.

## How it works

The TLS handshake is extended: the server requests a client certificate, the client presents one, the server validates it against a trusted CA. After the handshake, both sides know the other's identity. HTTP requests carry no auth header — the connection itself is authenticated.

```http
GET /internal/v1/users HTTP/1.1
Host: internal.example.com
# No Authorization header — identity is in the TLS layer
```

## When to use

- **Internal service-to-service** traffic, often via a service mesh (Istio, Linkerd) that handles certs automatically.
- **Partner integrations** with high security requirements where you control both ends.
- **Regulated industries** — Open Banking, healthcare, finance.
- **Zero-trust architectures** where network position grants no implicit trust.

## When not to use

- **Browser-facing APIs** — client certs in browsers are a UX disaster.
- **Public developer-facing APIs** — cert distribution and rotation is unworkable for arbitrary consumers; use [[API Keys]] or [[OAuth 2.0]].

## Flow

```
┌─────────┐                                       ┌─────────┐
│ Client  │                                       │ Server  │
└────┬────┘                                       └────┬────┘
     │  TLS ClientHello                                │
     │ ──────────────────────────────────────────────► │
     │                                                 │
     │  ServerHello                                    │
     │  Server Certificate                             │
     │  CertificateRequest ← (this is the mTLS bit)    │
     │ ◄────────────────────────────────────────────── │
     │                                                 │
     │  Client Certificate                             │
     │  CertificateVerify (proves key ownership)       │
     │  Finished                                       │
     │ ──────────────────────────────────────────────► │
     │                                                 │ 1. Verify cert chain 
     |                                                 |    against trusted CA
     │                                                 │ 2. Check revocation
     |                                                 |    (CRL/OCSP)
     │                                                 │ 3. Extract subject/SANs
     |                                                 |    as identity
     │  Finished                                       │
     │ ◄────────────────────────────────────────────── │
     │                                                 │
     │         ── encrypted application data ──        │
     │                                                 │
     │  GET /api/users                                 │
     │  (no Authorization header needed)               │
     │ ──────────────────────────────────────────────► │
     │                                                 │ 4. Look up identity → 
     |                                                 |    permissions
     │  200 OK + data                                  │
     │ ◄────────────────────────────────────────────── │
```

Identity is established **once per connection** during the handshake. Subsequent requests on the same connection inherit it — no per-request auth check is needed beyond authorization.

## Register / login / logout

These concepts don't map cleanly to mTLS — there's no session, no login form, no logout button. The closest analogs:

### Register (cert provisioning)

Done out of band, typically:

```
1. Client generates a key pair locally (never shares the private key)
2. Client creates a Certificate Signing Request (CSR):
       - public key
       - intended identity (subject CN / SANs)
3. CSR submitted to a Certificate Authority (CA)
4. CA verifies the request (manually, automated workflow, or
   automatic issuance in a service mesh like Istio's Citadel)
5. CA signs and returns a cert
6. Client stores cert + private key; presents on every connection
```

Service meshes automate steps 1–6 entirely — pods get certs at startup without any manual workflow.

### "Login"

The TLS handshake itself. Happens transparently every time a new connection is opened. There's no form, no credentials, no redirect — the client just presents its cert and the server validates it.

TLS supports **session resumption** (session tickets / pre-shared keys) so reconnecting skips the full handshake while reusing the original identity.

### "Logout"

There isn't one. Close the connection — that's it. The next connection re-authenticates from scratch. To revoke access:

- **Cert revocation:** add the cert serial to a Certificate Revocation List (CRL) or have OCSP responders report it as revoked. Requires the verifier to actually check revocation, which many systems silently skip.
- **Short cert lifetimes:** issue certs with hours-to-days validity instead of years. Compromised certs expire quickly without needing revocation infrastructure. Service meshes typically use 24-hour certs with automatic rotation.
- **Rotate the trust anchor:** in extreme cases, issue from a new CA and remove the old one from the trust store.

> [!tip] Short-lived certs > revocation
> Revocation checking is operationally messy (CRLs go stale, OCSP responders go down). Most modern setups skip it entirely and rely on short cert lifetimes for compromise mitigation.

## Authorization (what the cert holder can access)

The cert proves *who* the client is. Authorization is a separate decision based on that identity:

```
Cert subject: CN=billing-service.production.svc
       ↓
Identity: "billing-service in production"
       ↓
Policy lookup (server-side or service mesh policy):
   "billing-service may call /v1/invoices and /v1/payments"
       ↓
Allow or 403 Forbidden
```

How identity maps to permissions:

- **Subject Alternative Names (SANs)** — the modern way to encode identity. DNS names, URIs, or SPIFFE IDs.
- **SPIFFE IDs** (`spiffe://example.com/ns/prod/sa/billing`) — a workload-identity standard widely used in service meshes.
- **Policy lives outside the cert** — in a service mesh policy (Istio `AuthorizationPolicy`), an API gateway config, or your application code.

In Node.js, you extract the peer cert from the socket after the connection is established:

```typescript
app.use((req, res, next) => {
  const cert = (req.socket as TLSSocket).getPeerCertificate();
  const identity = cert.subject?.CN;       // or parse SANs
  req.identity = identity;
  next();
});
```

> [!tip] Don't put permissions inside the cert
> Tempting to encode scopes in cert extensions, but certs are hard to rotate. Keep the cert about *identity* only; let a separate policy system handle authorization decisions. This way changing what `billing-service` can do doesn't require reissuing its cert.

## Common mistakes

> [!warning] Anti-patterns
> - **Expired certs.** Rotation is operationally hard; outages from expired certs are routine (Microsoft, Spotify, GitHub have all had public ones). Build rotation tooling before you need it.
> - **Skipping revocation checks.** CRL and OCSP are messy; many systems silently don't check. Either check them or use short-lived certs.
> - **Weak ciphers / old TLS.** Enforce TLS 1.2 minimum, ideally 1.3. Disable weak cipher suites explicitly.
> - **Overly broad trust anchors.** Trusting a public CA means *any* cert that CA issues can authenticate. Use a private CA dedicated to your environment.
> - **Private key leakage.** Same blast radius as [[API Keys]] but much harder to rotate. Use HSMs or cloud KMS for high-value keys.
> - **`rejectUnauthorized: false` in production.** Disables cert validation entirely. Common in dev configs that accidentally ship to prod.
> - **No identity-to-application mapping plan.** "TLS says this is `client-xyz`" — now what does that mean for authorization? Decide before deploying.
> - **Putting permissions in cert extensions.** Couples authorization to cert lifecycle. Keep them separate.

## Resources for TypeScript implementation

mTLS in Node.js is built into the standard library. There's no real third-party library because there's no need — TLS lives below the framework layer.

**Built-in `https` / `tls` modules**
- *Why use:* nothing else is needed; Node's TLS stack is solid and well-tested.
- *Why not:* low-level; you wire up cert paths, CA chains, and reload-on-rotation logic yourself.
- *Handles automatically:* TLS handshake, cipher negotiation, certificate chain validation, hostname verification (when configured).
- *Be aware of:* set `requestCert: true` and `rejectUnauthorized: true` server-side, pass `ca`/`cert`/`key` correctly, handle cert rotation without restarting (or accept brief downtime), extract client identity via `req.socket.getPeerCertificate()` and pass it down to your app layer.

**Frameworks** (Express, Fastify, NestJS, Hono)
- *Why use:* you'll use one anyway for routing.
- *Why not:* mTLS is configured at the HTTP server level (where you call `https.createServer(...)`), not as application middleware. The framework is unaware.
- *Handles automatically:* nothing mTLS-specific.
- *Be aware of:* the framework receives requests *after* the TLS layer authenticates; write middleware that reads peer cert info from the socket and attaches identity to `req`.

**For dev/testing: `mkcert`**
- Generates locally-trusted certs for development. Saves you from `rejectUnauthorized: false` shortcuts that leak into prod.

**For prod cert lifecycle:**
- Kubernetes: **`cert-manager`** automates issuance and rotation.
- Service meshes (**Istio**, **Linkerd**) issue and rotate mesh-internal certs automatically with no app changes.
- **HashiCorp Vault PKI**, **Smallstep `step-ca`** for self-hosted CAs.

> [!tip] Use a service mesh if you can
> Hand-rolling mTLS at the application layer is painful. If you're running in Kubernetes, a service mesh (Istio, Linkerd) gives you mTLS between every pod for free, with automatic cert issuance and rotation.

## References

- Node.js TLS docs: https://nodejs.org/api/tls.html
- mkcert (dev certs): https://github.com/FiloSottile/mkcert
- cert-manager (Kubernetes): https://cert-manager.io/
- Smallstep `step-ca`: https://smallstep.com/docs/step-ca/
- SPIFFE workload identity: https://spiffe.io/

## Related

- [[API Keys]]
- [[OAuth 2.0]]
- [[JSON Web Tokens]]
- [[Authentication strategies compared]]
- [[Authentication and Authorization patterns]]
- [[Service mesh]]