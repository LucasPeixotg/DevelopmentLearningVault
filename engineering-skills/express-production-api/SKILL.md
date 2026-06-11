---
name: express-production-api
description: Use this skill when scaffolding or hardening a production Express + TypeScript API — choosing the middleware baseline, ordering it correctly, validating input, and wiring in security headers, CORS, rate limiting, logging, and the error handler. The glue layer that ties the other API skills together.
---

# Express production API baseline

## When to use this

You're starting a new Express + TypeScript API, or hardening one before production. Express
ships with almost no security or operational defaults — no security headers, no input
validation, no rate limiting, no compression. This is the standard baseline that fills the
gaps, in the order that matters.

## Decision

Use **Express 5** (stable since late 2024). Add the security trio + validation + logging +
compression. The non-negotiable rule is **ordering**: cheap rejections and security first,
expensive work last, the error handler wrapping everything.

- **`helmet`** — ~15 security headers in one line. The highest-impact single change. **First.**
- **`cors`** — explicit origin **allowlist**; never reflect `Origin` blindly.
- **`express-rate-limit`** — high in the stack, before auth, to blunt brute-force/DoS
  (see `api-rate-limiting`; use a Redis store multi-server).
- **`compression`** — gzip/brotli; meaningful for JSON-heavy APIs.
- **`pino` / `pino-http`** — structured JSON logging (queryable in an aggregator).
- **Input validation** — **`zod`** (TS-first, infers types) is the modern default; `Joi` or
  `express-validator` also fine. Not optional: ~half of web vulns trace to bad input
  validation, and the frontend is not a trust boundary.
- **Error handler** — RFC 9457 `application/problem+json` (see `rest-api-error-design`). **Last.**

## Implementation

```typescript
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';
import compression from 'compression';
import pinoHttp from 'pino-http';
import { randomUUID } from 'crypto';

const app = express();

app.use(helmet());                                       // 1. security headers FIRST
app.use(cors({ origin: ['https://app.example.com'] }));  // 2. explicit allowlist, not `true`
app.use(rateLimit({ windowMs: 60_000, max: 200,
                    standardHeaders: 'draft-7', legacyHeaders: false }));  // 3. shed abuse early
app.use(compression());                                  // 4. compress responses
app.use(pinoHttp());                                     // 5. structured request logging
app.use((req, _res, next) => {                           // 6. correlation id
  (req as any).id = req.headers['x-request-id'] ?? randomUUID(); next();
});
app.use(express.json());                                 // 7. built-in body parsing (no body-parser)

// ... routes (validate input with zod inside handlers) ...

app.use(errorHandler);                                   // LAST: problem+json, logs full detail
```

### Why the ordering

```
helmet        → even error responses from early-failing routes carry security headers
cors          → reject disallowed origins before doing work
rate limiting → shed abusive traffic before parsing or auth
compression   → compress whatever routes return
logging       → capture every request (+ correlation id)
body parsing  → parse only requests that got this far
routes        → your handlers
error handler → catches everything, formats as problem+json
```

### Express 5 — stop installing these

- **`body-parser`** — built in; use `express.json()` / `express.urlencoded()`.
- **`express-async-errors`** — Express 5 natively forwards rejected async-handler promises
  to the error middleware. The shim is obsolete.

### CSRF

- **Bearer-token APIs** (no cookies) generally don't need CSRF protection — no ambient credential.
- **Cookie-based sessions** → `SameSite=Lax`/`Strict` + double-submit via the maintained
  **`csrf-csrf`** package. **`csurf` is deprecated and archived — do not use it.**

### HATEOAS (when resource actions depend on state)

If valid operations depend on resource state (order `pending` vs `shipped`), have the server
return the allowed next actions as **links** in the response, so clients discover the state
machine instead of hard-coding it:

```json
{ "id": "ord_42", "status": "pending",
  "_links": { "self": { "href": "/v1/orders/ord_42" },
              "cancel": { "href": "/v1/orders/ord_42/cancel", "method": "POST" } } }
```

## Checklist

- [ ] `helmet` first; CSP customised and tested in report-only before enforcing.
- [ ] `cors` uses an explicit allowlist (never `origin: true` in production).
- [ ] Rate limiter high in the stack, before auth; Redis store if multi-server.
- [ ] Input validated (zod) on every handler that takes a body/params.
- [ ] Structured logging (pino) with a correlation id attached early.
- [ ] Error handler registered **last**, emitting `application/problem+json`.
- [ ] No `body-parser` / `express-async-errors` on Express 5; no `csurf`.

## Pitfalls

- **Helmet after routes** — early-failing routes send unprotected headers.
- **`cors({ origin: true })`** in production — reflects any origin, disabling the protection.
- **Skipping validation** "because the frontend validates" — the frontend isn't a trust boundary.
- **Still installing `body-parser` / `express-async-errors`** on Express 5 — redundant.
- **Using `csurf`** — deprecated; use `csrf-csrf` or `SameSite` cookies.
- **Unstructured `console.log`** — not queryable in a log aggregator.

## Reference

Vault notes:
- `API Best Practices/Express production middleware baseline.md`
- `API Best Practices/CORS.md`
- `API Best Practices/Hypermedia and HATEOAS.md`

External: Express security best practices
(https://expressjs.com/en/advanced/best-practice-security/) ·
helmet (https://github.com/helmetjs/helmet) · zod (https://zod.dev/) ·
pino (https://getpino.io/) · Express 5 migration (https://expressjs.com/en/guide/migrating-5.html)

Related skills: `rest-api-error-design` (the error handler this wires in last),
`api-rate-limiting` (the limiter middleware), `api-authentication` (auth middleware placement).
