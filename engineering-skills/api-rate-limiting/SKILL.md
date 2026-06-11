---
name: api-rate-limiting
description: Use this skill when protecting an API from abusive or runaway clients — adding rate limiting/throttling, picking an algorithm and store, setting the right headers, or designing per-plan/per-endpoint tiers. Produces limits that scale across servers and let clients back off gracefully.
---

# API rate limiting

## When to use this

You need to cap how many requests a client can make: protecting the server from a buggy
retry loop, scraper, or DoS; enforcing fair use; or monetising access tiers. Limits added
after launch can break clients who relied on having none — design them early.

## Decision

### Algorithm

| Algorithm | Burst-friendly | Accuracy | Use for |
|---|---|---|---|
| Fixed window | no (boundary burst) | low | simple internal limits |
| Sliding window (counter) | somewhat | ~exact | **most public APIs** (default) |
| Token bucket | yes (by design) | exact | consumer/cloud APIs needing bursts |
| Leaky bucket | no (queues) | n/a | internal traffic shaping |

**Default to sliding-window counter** — no boundary burst, low memory, near-exact. Use
token bucket when legitimate bursts must be allowed.

### Store / approach (Node + Express)

| Approach | Multi-server correct | Use when |
|---|---|---|
| In-process (memory) | **No** (`max × N` pods) | prototype / single server / local dev only |
| `express-rate-limit` + `rate-limit-redis` | Yes | **multi-server, standard cases** (default) |
| Custom Redis + Lua sliding window | Yes | multi-server needing custom plan-tier logic |
| API gateway (Kong, AWS) | Yes | scale / DDoS / infra layer |

**Never use in-process in multi-server production** — with N pods a 100 req/min limit
becomes 100×N. Most production APIs **layer two**: gateway for coarse/DDoS protection,
app middleware (Redis) for per-user/per-plan/per-endpoint business rules that need your DB
and auth context.

### Key the limit on identity, not just IP

Anonymous → IP (coarse, spoofable); authenticated → API key / OAuth `client_id`; per user
→ JWT `sub`; per org → account id. IP-only limiting harms shared-NAT users and is easily
evaded.

## Implementation

### Standard headers — send on EVERY response

Prefer the IETF draft `RateLimit-*` (unambiguous: `Reset` is seconds-until-reset), not the
legacy `X-RateLimit-*` (ambiguous `Reset`). Clients need these on successes to back off
*before* hitting the wall.

```http
RateLimit-Limit: 1000
RateLimit-Remaining: 943
RateLimit-Reset: 57          # seconds until reset
```

### On rejection: 429 + Retry-After + problem+json

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Remaining: 0
RateLimit-Reset: 30
Content-Type: application/problem+json

{ "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate limit exceeded", "status": 429,
  "detail": "Limit is 1000 req/hour. Retry in 30s." }
```

Without `Retry-After`, clients hot-loop and make it worse.

### express-rate-limit + Redis (the standard multi-server setup)

```typescript
import { rateLimit } from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';

app.use(rateLimit({
  windowMs: 60_000,
  max: 200,
  standardHeaders: 'draft-7',   // emits RateLimit-* automatically
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redis.call(...args) }),
  keyGenerator: (req) => req.user?.id ?? req.ip,   // per-user when authed
}));
```

### Tier by plan and by endpoint cost

```
Free:    100 req/day,   10 req/min        GET  /v1/users/*  → 1,000 req/min (cheap)
Starter: 5,000/day,    100/min            POST /v1/charges  →   100 req/min (write)
Pro:    50,000/day,  1,000/min            POST /v1/reports  →    10 req/min (expensive)
```

Stripe-style multiple simultaneous windows (per-second + per-minute + per-hour) stop
clients spreading bursts to evade the hourly cap.

## Checklist

- [ ] `RateLimit-*` headers on **every** response (not only `429`).
- [ ] `429` always includes `Retry-After`; body is `application/problem+json`.
- [ ] Shared store (Redis) in multi-server deployments — no per-pod counters.
- [ ] Keyed on identity (API key / `sub` / account), not IP alone, for authed traffic.
- [ ] Per-endpoint limits — cheap reads ≠ expensive writes/exports.
- [ ] Exemptions for internal health checks / monitoring.
- [ ] Limited at both the gateway and the app layer for anything public.

## Pitfalls

- **No `Retry-After` on 429** — clients hot-loop.
- **In-process limiter across multiple pods** — effective limit = `max × pods`.
- **App-layer only** — you've already paid TLS/auth cost; also limit at the gateway.
- **One limit for all endpoints** — `POST /export` shouldn't share with `GET /ping`.
- **IP-only limiting** — punishes shared NAT, trivially evaded by rotating IPs.
- **No headers on success** — clients can't implement backpressure.
- **Legacy `X-RateLimit-Reset`** ambiguity (timestamp vs seconds) — use `RateLimit-*`.

## Reference

Vault notes:
- `API Best Practices/Rate Limiting & Throttling/Rate limiting & throttling.md`
- `API Best Practices/Rate Limiting & Throttling/Rate Limiting in Node and Typescript Comparison.md`
- `API Best Practices/Rate Limiting & Throttling/In-process Rate Limiting Implementation.md`
- `API Best Practices/Rate Limiting & Throttling/Redis Libraries Rate Limiting Implementation.md`
- `API Best Practices/Rate Limiting & Throttling/Redis-backed sliding window rate limiting Implementation.md`
- `API Best Practices/Rate Limiting & Throttling/API Gateway Rate Limiting Implementation.md`

External: draft-ietf-httpapi-ratelimit-headers
(https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) ·
Cloudflare sliding window (https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) ·
Stripe rate limits (https://stripe.com/docs/rate-limits)

Related skills: `rest-api-error-design` (429 problem body), `api-authentication`
(the identity you key limits on), `express-production-api` (where the limiter sits in the stack).
