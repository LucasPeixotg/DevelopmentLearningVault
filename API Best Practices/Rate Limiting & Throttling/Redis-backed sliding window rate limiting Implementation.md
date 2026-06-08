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

# Redis-backed sliding window rate limiting Implementation

> Summary: Implements the approximate sliding window counter algorithm using Redis as the shared store. Works correctly across any number of server instances. A Lua script executes the read-increment-expire sequence atomically on the Redis server, eliminating race conditions. The production-ready approach when you need custom logic (per-plan tiers, per-endpoint costs) that a library or API gateway can't express. Requires `ioredis`.

## When to use

- Multi-server production deployments
- You need fine-grained control — per-plan limits looked up from your own database, per-endpoint cost factors, custom key strategies
- You want to understand what's happening rather than rely on a library

If you don't need custom logic, [[Redis Libraries Rate Limiting Implementation]] is faster to ship.

## How the algorithm works

Two Redis keys per client per window — current window and previous window. On each request:

```
approximateCount = previousCount × (1 - elapsedFraction) + currentCount
```

If you're 30% through the current window, previous traffic is weighted at 70%. Near-exact accuracy without storing every timestamp. This is the same approach Cloudflare uses.

## Implementation

```bash
npm install ioredis
```

```typescript
// redisRateLimit.ts
import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL ?? 'redis://localhost:6379');

function getClientKey(req: Request): string {
  const userId = (req as any).user?.id;
  return userId ? `user:${userId}` : `ip:${req.ip}`;
}

// Lua runs atomically on Redis — no race condition between GET and INCR
const slidingWindowScript = `
  local currentKey  = KEYS[1]
  local previousKey = KEYS[2]
  local limit       = tonumber(ARGV[1])
  local windowMs    = tonumber(ARGV[2])
  local now         = tonumber(ARGV[3])

  local elapsed  = (now % windowMs) / windowMs
  local previous = tonumber(redis.call('GET', previousKey)) or 0
  local current  = tonumber(redis.call('GET', currentKey))  or 0
  local approx   = math.floor(previous * (1 - elapsed) + current)

  if approx >= limit then
    return { -1, current, previous }
  end

  redis.call('INCR', currentKey)
  redis.call('EXPIRE', currentKey, math.ceil(windowMs / 1000) * 2)
  return { limit - approx - 1, current + 1, previous }
`;

export function redisRateLimit(options: {
  windowSeconds: number;
  maxRequests: number;
  keyPrefix?: string;
}) {
  const windowMs = options.windowSeconds * 1000;

  return async (req: Request, res: Response, next: NextFunction) => {
    const clientKey = getClientKey(req);
    const prefix = options.keyPrefix ?? 'rl';
    const now = Date.now();
    const currentWindow = Math.floor(now / windowMs);

    const currentKey  = `${prefix}:${clientKey}:${currentWindow}`;
    const previousKey = `${prefix}:${clientKey}:${currentWindow - 1}`;
    const resetIn = Math.ceil(options.windowSeconds - (now % windowMs) / 1000);

    try {
      const [remaining] = await redis.eval(
        slidingWindowScript,
        2,
        currentKey,
        previousKey,
        options.maxRequests,
        windowMs,
        now
      ) as number[];

      res.setHeader('RateLimit-Limit', options.maxRequests);
      res.setHeader('RateLimit-Remaining', Math.max(0, remaining));
      res.setHeader('RateLimit-Reset', resetIn);

      if (remaining < 0) {
        res.setHeader('Retry-After', resetIn);
        res.status(429).json({
          type: 'https://api.example.com/errors/rate-limit-exceeded',
          title: 'Rate limit exceeded',
          status: 429,
          detail: `Limit is ${options.maxRequests} req/${options.windowSeconds}s. Retry in ${resetIn}s.`,
        });
        return;
      }

      next();
    } catch (err) {
      // Fail open — allow the request if Redis is unavailable.
      // Log loudly; page on this.
      console.error('[rate-limiter] Redis error:', err);
      next();
    }
  };
}
```

```typescript
// app.ts
import express from 'express';
import { redisRateLimit } from './redisRateLimit';
import { authenticate } from './auth';

const app = express();

// Global limit — runs before auth so also protects /login
app.use(redisRateLimit({ windowSeconds: 60, maxRequests: 200 }));

app.use(authenticate); // sets req.user — key switches from IP to user ID

app.post(
  '/v1/charges',
  redisRateLimit({ windowSeconds: 60, maxRequests: 50,  keyPrefix: 'rl:charges' }),
  chargesHandler
);

app.post(
  '/v1/reports',
  redisRateLimit({ windowSeconds: 60, maxRequests: 5,   keyPrefix: 'rl:reports' }),
  reportsHandler
);
```

## Key design decisions

> [!tip] Why Lua?
> Without the Lua script, two simultaneous requests can both read `0` and both pass the limit check before either increments the counter. The Lua script runs as a single atomic operation on Redis — no race condition is possible.

> [!tip] Fail open on Redis errors
> When Redis goes down, requests are allowed through rather than blocking the entire API. A Redis outage shouldn't cause an API outage. Log it loudly and alert. The alternative — fail closed — turns every Redis blip into a customer-facing incident.

> [!tip] Place global limit before auth middleware
> IP-based limiting catches unauthenticated traffic (bots, login brute force) before auth runs. After auth, `getClientKey` switches to user ID for finer-grained per-user enforcement.

## Pros and cons

**Pros:**
- Correct across any number of server instances.
- Atomic — no race conditions.
- Survives server restarts.
- Full control over key strategy, algorithm, and per-endpoint configuration.
- Approximate sliding window avoids the boundary burst of fixed window.

**Cons:**
- Redis is a required dependency — adds operational overhead.
- Every request incurs a Redis round-trip (~1ms locally, more cross-AZ).
- More code to own; bugs are yours to fix.
- Lua debugging is painful.

## Common mistakes

- **Not using Lua.** A plain `GET` + `INCR` in application code has a race condition. Always use an atomic script or a Redis transaction.
- **Setting `EXPIRE` too short.** If the previous window's key expires before you read it, the sliding window calculation becomes a fixed window. Set `EXPIRE` to at least `windowSeconds × 2`.
- **Single Redis instance without a fallback.** If Redis goes down and you fail closed, so does your API. Fail open and alert.
- **Not scoping keys by endpoint.** Without `keyPrefix`, a global and per-endpoint limiter share the same counter, and one can starve the other.

## References

- `ioredis`: https://github.com/redis/ioredis
- Cloudflare sliding window counter: https://blog.cloudflare.com/counting-things-a-lot-of-different-things/
- Redis Lua scripting: https://redis.io/docs/manual/programmability/lua-api/

## Related

- [[Rate limiting & throttling]]
- [[In-process Rate Limiting Implementation]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[API Gateway Rate Limiting Implementation]]
- [[Rate Limiting in Node and Typescript Comparison]]
