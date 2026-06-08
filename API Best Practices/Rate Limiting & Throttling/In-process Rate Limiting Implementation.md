---
tags:
  - api-design
  - typescript
  - nodejs
  - rate-limiting
  - implementation
created: 2026-06-03
implemented: false
---
# In-process Rate Limiting Implementation

> Summary: Stores rate limit counters in application memory using a `Map`. Zero external dependencies — the simplest possible approach. Only correct for single-server deployments; breaks immediately across multiple instances because each server has its own counter. Use this for local development and prototyping; use [[Redis-backed sliding window rate limiting Implementation]] or [[Redis Libraries Rate Limiting Implementation]] in production.

## When to use

- Local development
- Single-server deployments where simplicity beats correctness guarantees
- Tests (swap this out for the Redis implementation in production)

## Implementation

```typescript
// rateLimit.ts
import { Request, Response, NextFunction } from 'express';

interface WindowEntry {
  count: number;
  resetAt: number;
}

// In-memory store — does not survive restarts
const store = new Map<string, WindowEntry>();

function getClientKey(req: Request): string {
  // Prefer authenticated identity over IP when available
  const userId = (req as any).user?.id;
  return userId ? `user:${userId}` : `ip:${req.ip}`;
}

export function rateLimit(options: {
  windowSeconds: number;
  maxRequests: number;
}) {
  return (req: Request, res: Response, next: NextFunction) => {
    const key = getClientKey(req);
    const now = Date.now();
    const windowMs = options.windowSeconds * 1000;

    let entry = store.get(key);

    if (!entry || now >= entry.resetAt) {
      entry = { count: 0, resetAt: now + windowMs };
      store.set(key, entry);
    }

    entry.count++;

    const remaining = Math.max(0, options.maxRequests - entry.count);
    const resetIn = Math.ceil((entry.resetAt - now) / 1000);

    res.setHeader('RateLimit-Limit', options.maxRequests);
    res.setHeader('RateLimit-Remaining', remaining);
    res.setHeader('RateLimit-Reset', resetIn);

    if (entry.count > options.maxRequests) {
      res.setHeader('Retry-After', resetIn);
      res.status(429).json({
        type: 'https://api.example.com/errors/rate-limit-exceeded',
        title: 'Rate limit exceeded',
        status: 429,
        detail: `Limit is ${options.maxRequests} requests per ${options.windowSeconds}s. Retry in ${resetIn}s.`,
      });
      return;
    }

    next();
  };
}
```

```typescript
// app.ts
import express from 'express';
import { rateLimit } from './rateLimit';

const app = express();

// Global baseline limit
app.use(rateLimit({ windowSeconds: 60, maxRequests: 100 }));

// Stricter limit on an expensive endpoint
app.post(
  '/v1/reports',
  rateLimit({ windowSeconds: 60, maxRequests: 10 }),
  reportsHandler
);
```

## How it works

This is a **fixed window** implementation. On every request:

1. Derive a key from user ID (if authenticated) or IP address.
2. Check if a window entry exists and hasn't expired; create one if not.
3. Increment the counter.
4. Set `RateLimit-*` headers on every response.
5. Reject with `429` if the counter exceeds the limit.

## Pros and cons

**Pros:**
- Zero dependencies — nothing to install, configure, or operate.
- Trivially easy to understand and debug.
- Works perfectly for a single server.

**Cons:**
- **Memory leak.** The `store` map grows forever with unique IPs/users. Needs a periodic cleanup job or an LRU map (e.g. `mnemonist/lru-map`).
- **Does not survive restarts.** Counters reset on every deploy — a client can exhaust their limit, wait for a deploy, and reset.
- **Broken across multiple servers.** Two instances = double the effective limit per client. This is the critical failure mode.
- **Fixed window boundary burst.** A client can make 2× the limit in 2 seconds by straddling a window reset.

> [!warning] Never use this with more than one server instance
> Each pod has its own counter. With 3 pods, the effective rate limit is `maxRequests × 3`. Use the [[Redis-backed sliding window rate limiting Implementation]] for multi-server deployments.

## Common mistakes

- **Forgetting the cleanup.** Without eviction, long-running servers accumulate millions of entries. Use an LRU map or a scheduled `store.clear()` during low traffic.
- **Keying only on IP.** Users behind shared NAT (offices, universities) share an IP. Switch to user ID as soon as auth is available.
- **Using this in staging with multiple replicas.** Staging often mimics production topology — the bug won't surface in single-instance dev.

## References

- `mnemonist/lru-map` for bounded in-memory stores: https://yomguithereal.github.io/mnemonist/lru-map
- [[Rate limiting & throttling]] — algorithm theory and header standards

## Related

- [[Rate limiting & throttling]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[API Gateway Rate Limiting Implementation]]
- [[Rate Limiting in Node and Typescript Comparison]]
