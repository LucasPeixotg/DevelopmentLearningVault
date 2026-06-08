---
tags:
  - api-design
  - typescript
  - nodejs
  - rate-limiting
  - redis
  - implementation
created: 2026-06-03
implemented: false
---

# Redis Libraries Rate Limiting Implementation

> Summary: Uses `express-rate-limit` with `rate-limit-redis` as the backing store. The pragmatic middle ground — a well-maintained library handles edge cases (concurrent requests, Redis reconnection, header formatting) while Redis provides shared state across instances. Less code to own than the [[Redis-backed sliding window rate limiting Implementation]], with slightly less control. The recommended starting point for most production Express APIs.

## When to use

- Multi-server production deployments
- You want Redis-backed correctness without writing the algorithm yourself
- You're happy with the library's algorithm choices and header format
- You want `standardHeaders: 'draft-7'` (IETF `RateLimit-*` headers) for free

Use [[Redis-backed sliding window rate limiting Implementation]] instead when you need custom logic the library can't express (plan-tier lookups from your database, multi-dimensional limits, custom cost-per-endpoint).

## Installation

```bash
npm install express-rate-limit rate-limit-redis ioredis
```

## Implementation

```typescript
// app.ts
import express from 'express';
import { rateLimit } from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL ?? 'redis://localhost:6379');
const app = express();

function makeStore(prefix: string) {
  return new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
    prefix,
  });
}

// ── Global limit ────────────────────────────────────────────────────────────
app.use(
  rateLimit({
    windowMs: 60 * 1000,
    max: 200,
    standardHeaders: 'draft-7',  // emit IETF RateLimit-* headers
    legacyHeaders: false,          // suppress X-RateLimit-* headers
    store: makeStore('rl:global'),
    keyGenerator: (req) => (req as any).user?.id ?? req.ip ?? 'unknown',
    handler: (req, res, _next, options) => {
      res.status(429).json({
        type: 'https://api.example.com/errors/rate-limit-exceeded',
        title: 'Rate limit exceeded',
        status: 429,
        detail: `Limit is ${options.max} req/min. Retry after ${Math.ceil(options.windowMs / 1000)}s.`,
      });
    },
  })
);

// ── Per-endpoint stricter limit ──────────────────────────────────────────────
app.post(
  '/v1/reports',
  rateLimit({
    windowMs: 60 * 1000,
    max: 5,
    standardHeaders: 'draft-7',
    legacyHeaders: false,
    store: makeStore('rl:reports'),
    keyGenerator: (req) => (req as any).user?.id ?? req.ip ?? 'unknown',
  }),
  reportsHandler
);
```

## Key options

| Option | What it does |
|---|---|
| `windowMs` | Window length in milliseconds |
| `max` | Maximum requests per window |
| `standardHeaders: 'draft-7'` | Emit IETF `RateLimit-*` headers automatically |
| `legacyHeaders: false` | Suppress `X-RateLimit-*` headers — don't emit both |
| `store` | Where counters live — `RedisStore` for multi-server |
| `keyGenerator` | How to identify a client — prefer user ID over IP |
| `handler` | Custom response when limit is exceeded |
| `skip` | Function — return `true` to exempt a request (e.g. internal health checks) |

## Skipping internal traffic

```typescript
rateLimit({
  // ...
  skip: (req) => {
    // Exempt internal health checks and monitoring
    return req.path === '/health' || req.ip === '10.0.0.1';
  },
})
```

## Testing with the in-memory store

Swap the Redis store for the default in-memory store in tests — no Redis needed:

```typescript
// rateLimit.config.ts
import { rateLimit } from 'express-rate-limit';

export function makeRateLimit(max: number) {
  return rateLimit({
    windowMs: 60 * 1000,
    max,
    standardHeaders: 'draft-7',
    legacyHeaders: false,
    // No store → uses in-memory (fine for tests, not for production)
  });
}
```

## Pros and cons

**Pros:**
- Much less code to own — edge cases and header formatting are handled.
- `standardHeaders: 'draft-7'` emits correct IETF headers with one option.
- Easy to swap the store (in-memory for tests, Redis for production).
- Active maintenance — bugs are fixed upstream.
- `skip` makes exempting internal traffic trivial.

**Cons:**
- Less control over the exact algorithm — fixed window by default (Redis store switches to sliding window, but the behaviour isn't always obvious).
- One more library dependency to audit and update.
- Custom logic (plan-tier lookups, cost-per-endpoint factors) requires awkward workarounds.
- Slightly less transparent — debugging requires understanding the library internals.

> [!warning] Check which algorithm your store uses
> `express-rate-limit` defaults to fixed window with the in-memory store. `rate-limit-redis` uses a fixed window too unless you configure it otherwise. The boundary burst problem is real — test for it if burst behaviour matters for your use case. For sliding window, use [[Redis-backed sliding window rate limiting Implementation]].

## Common mistakes

- **Emitting both `standardHeaders` and `legacyHeaders`.** Clients get confused by two conflicting sets of headers. Pick one.
- **No custom `keyGenerator`.** The default keys on IP, which breaks under shared NAT. Always prefer user ID.
- **No `skip` for internal traffic.** Health checks and monitoring eat into quota.
- **Reusing the same store prefix for different limits.** `makeStore('rl:global')` and `makeStore('rl:reports')` must have different prefixes or they share counters.

## References

- `express-rate-limit`: https://github.com/express-rate-limit/express-rate-limit
- `rate-limit-redis`: https://github.com/express-rate-limit/rate-limit-redis
- `ioredis`: https://github.com/redis/ioredis
- IETF `RateLimit` headers draft: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/

## Related

- [[Rate limiting & throttling]]
- [[In-process Rate Limiting Implementation]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[API Gateway Rate Limiting Implementation]]
- [[Rate Limiting in Node and Typescript Comparison]]
