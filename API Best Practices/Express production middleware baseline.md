---
tags: [nodejs, express, typescript, security, best-practices]
created: 2026-06-03
---
# Express production middleware baseline

> Summary: Express ships with almost no security or operational defaults — no security headers, no input validation, no rate limiting, no compression. A small standard toolkit fills the gaps: `helmet` (headers), `cors` (cross-origin), `express-rate-limit` (abuse), a validation library, structured logging, and `compression`. Ordering matters — security headers first so even error responses are protected, error handler last. Express 5 (stable since late 2024) made `body-parser` and `express-async-errors` obsolete.

## The security trio

Not alternatives — use all three together. Each covers a different attack vector.

**`helmet`** — sets ~15 HTTP security headers (CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`, etc.) in one line.
- *Highest-impact single change* you can make to an Express app.
- Apply it **first**, before any other middleware. If registered after routes, a route that errors before reaching it sends unprotected headers.
- Defaults are good, but customise Content-Security-Policy — it's the strongest XSS protection and the most likely to break functionality. Test in report-only mode before enforcing.

**`cors`** — implements Cross-Origin Resource Sharing (see [[CORS]]).
- Maintain an explicit origin allowlist; never reflect the `Origin` header blindly.
- Wildcard `*` and credentials are mutually exclusive — the browser rejects that combination.

**`express-rate-limit`** — limits requests per client (see [[Rate limiting & throttling]]).
- Place high in the stack, before auth routes, to blunt brute-force and DoS.
- Use a Redis store in multi-server deployments (see [[Redis Libraries Rate Limiting Implementation]]).

## Input validation

Express does no validation out of the box. Pick one:

| Library | Notes |
|---|---|
| **`zod`** | TypeScript-first; infers types from schemas so validation and types stay in sync. The modern default in TS codebases. |
| **`Joi`** | Mature, framework-agnostic schema validation. |
| **`express-validator`** | Middleware-style, wraps validator.js; fits naturally into the request pipeline. |

Roughly half of web vulnerabilities trace back to improper input validation — this is not optional.

## Logging

| Library | Notes |
|---|---|
| **`pino`** / **`pino-http`** | Fast, structured (JSON) logging. Modern default. Pairs with the correlation-ID pattern in [[API error design guidelines]]. |
| **`morgan`** | Simple HTTP request logging, good for development. |
| **`winston`** | Established structured logging with flexible transports. |

## Performance

**`compression`** — gzip/brotli response compression. One line, meaningful bandwidth savings for JSON-heavy APIs.

## Cookies & config

- **`cookie-parser`** — only if using cookies (session auth, CSRF tokens).
- **`dotenv`** — environment variables. Note: Node.js now has a built-in `--env-file` flag, so this may be unnecessary.

## CSRF — a caveat

> [!warning] `csurf` is deprecated and archived
> Don't use the old standard. Modern approaches:
> - **Bearer-token APIs** (not cookies) mostly don't need CSRF protection — there's no ambient credential for an attacker to ride on.
> - **Cookie-based sessions** → `SameSite=Lax` or `Strict` cookies plus the double-submit pattern via the maintained **`csrf-csrf`** package.

## What Express 5 made obsolete

> [!tip] Don't install these anymore
> - **`body-parser`** — built in. Use `express.json()` and `express.urlencoded()` directly.
> - **`express-async-errors`** — Express 5 natively catches rejected promises from async route handlers and forwards them to the error middleware. In Express 4, an unhandled async rejection would hang the request; in 5 it doesn't, so the shim is obsolete.

## Baseline setup

```typescript
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { rateLimit } from 'express-rate-limit';
import compression from 'compression';
import pinoHttp from 'pino-http';

const app = express();

app.use(helmet());                                   // 1. security headers FIRST
app.use(cors({ origin: ['https://app.example.com'] }));
app.use(rateLimit({ windowMs: 60_000, max: 200 }));
app.use(compression());
app.use(pinoHttp());                                 // structured request logging
app.use(express.json());                             // built-in body parsing

// ... routes ...

app.use(errorHandler);                               // RFC 7807 handler LAST
```

## Why the ordering matters

```
helmet        → so even error responses carry security headers
cors          → reject disallowed origins before doing work
rate limiting → shed abusive traffic before parsing or auth
compression   → compress whatever the routes return
logging       → capture every request
body parsing  → parse only requests that got this far
routes        → your handlers
error handler → catches everything, formats as problem+json
```

Security and cheap rejections go first; expensive work (parsing, route logic) goes last; the error handler wraps the whole thing.

## Common mistakes

> [!warning] Anti-patterns
> - **Helmet after routes.** Error responses from early-failing routes go out unprotected. Helmet first, always.
> - **`cors({ origin: true })` in production.** Reflects any origin — effectively disables the protection. Use an allowlist.
> - **Skipping input validation** because "the frontend validates." The frontend is not a trust boundary.
> - **Still installing `body-parser` / `express-async-errors`** on Express 5. Both are redundant.
> - **Using `csurf`.** Deprecated. Use `csrf-csrf` or `SameSite` cookies.
> - **No structured logging.** Unstructured `console.log` can't be queried in a log aggregator. Use `pino`.

## References

- Express production security best practices: https://expressjs.com/en/advanced/best-practice-security/
- helmet: https://github.com/helmetjs/helmet
- cors: https://github.com/expressjs/cors
- express-rate-limit: https://github.com/express-rate-limit/express-rate-limit
- zod: https://zod.dev/
- pino: https://getpino.io/
- Express 5 migration guide: https://expressjs.com/en/guide/migrating-5.html
- csrf-csrf: https://github.com/Psifi-Solutions/csrf-csrf

## Related

- [[CORS]]
- [[Rate limiting & throttling]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[API error design guidelines]]
- [[Error response formats]]
- [[Authentication strategies compared]]
