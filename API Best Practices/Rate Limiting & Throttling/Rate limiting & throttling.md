---
tags: [api-design, performance, security, rate-limiting, best-practices]
created: 2026-06-03
---
# Rate limiting & throttling

> Summary: Rate limiting caps how many requests a client can make in a time window — protecting the server, enforcing fair use, and enabling tiered plans. Throttling is the softer version: queue or slow requests rather than reject them. The four main algorithms are fixed window, sliding window, token bucket, and leaky bucket. The key wire-protocol concern is communicating limits via response headers so clients can back off gracefully.

## Why it exists

A single misbehaving client — a buggy retry loop, a scraper, a DDoS — can degrade your API for everyone. Rate limiting protects the server, enforces fairness, and is the primary lever for monetising access tiers.

- **Rate limiting** — hard cap. Exceed it, get `429 Too Many Requests`.
- **Throttling** — soft cap. Queue or delay requests rather than always hard-rejecting. The terms are used interchangeably in most APIs.

## The algorithms

| Algorithm | Burst-friendly | Memory | Accuracy | Best for |
|---|---|---|---|---|
| Fixed window | No (boundary burst) | Very low | Low | Simple internal limits |
| Sliding window (log) | No | High | Exact | Low-volume, high-accuracy |
| Sliding window (counter) | Somewhat | Low | ~Exact | **Most public APIs** |
| Token bucket | Yes (by design) | Low | Exact | **Consumer APIs, cloud APIs** |
| Leaky bucket | No (queues bursts) | Medium | N/A | Internal traffic shaping |

**Fixed window** — divide time into buckets, count per bucket. Simple, but a client can make 2× the limit in 2 seconds by straddling a window boundary.

**Sliding window (counter)** — approximate using two consecutive fixed windows, weighted by elapsed time. Low memory, near-exact, used by Cloudflare and most Redis-backed implementations.

**Token bucket** — bucket fills at a fixed rate up to a max capacity. Requests consume tokens. Allows legitimate burst traffic. Used by Stripe, AWS.

**Leaky bucket** — requests enter a queue, drain at a fixed rate. Smoothest output; adds latency; used for internal traffic shaping, not consumer APIs.

## Standard response headers

### Legacy (widely deployed, no official standard)

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 943
X-RateLimit-Reset: 1735693200   ← ambiguous: Unix timestamp or seconds?
```

### IETF draft (`RateLimit-*`) — the modern approach

```http
RateLimit-Limit: 1000
RateLimit-Remaining: 943
RateLimit-Reset: 57             ← always seconds until reset
```

Always send these on **every response**, not only on `429` — clients need them to implement backpressure before hitting the wall.

### On rejection: `429` + `Retry-After`

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 30
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Limit is 1000 req/hour. Retry in 30s."
}
```

> [!warning] Always include `Retry-After` on 429
> Without it, clients don't know when to retry and immediately hammer the same limit in a hot loop, making the problem worse.

## Designing rate limit tiers

### By identity

| Client type | Key on |
|---|---|
| Anonymous | IP address (coarse, easy to spoof) |
| Authenticated | API key or OAuth `client_id` |
| Per user | `sub` claim from JWT |
| Per organisation | Account ID |

### By plan

```
Free:       100 req/day,    10 req/min
Starter:  5,000 req/day,   100 req/min
Pro:     50,000 req/day, 1,000 req/min
```

### By endpoint

Not all endpoints are equally expensive:

```
GET  /v1/users/*      → 1,000 req/min   (cheap, cacheable)
POST /v1/charges      →   100 req/min   (write)
POST /v1/reports      →    10 req/min   (expensive compute)
```

### By time horizon (Stripe-style)

Multiple simultaneous windows prevent spreading bursts to evade hourly limits:

```http
X-RateLimit-Limit-Second: 25
X-RateLimit-Limit-Minute: 100
X-RateLimit-Limit-Hour:   1000
```

## Common mistakes

> [!warning] Anti-patterns
> - **No `Retry-After` on 429** — clients hot-loop.
> - **Rate limiting only at the app layer** — by then you've already paid the TLS/auth cost. Also limit at the gateway.
> - **Same limit for all endpoints** — `POST /export` and `GET /ping` should not share a limit.
> - **IP-only limiting** — harms shared NAT users; easily evaded with rotating IPs.
> - **Not sending headers on successful responses** — clients can't implement backpressure.
> - **Shared limits without a central store** — multiple servers each have their own counter, multiplying the effective limit.
> - **No exemptions for internal services** — monitoring and health checks eat into quota.

## References

- RFC 9110 §15.5.29 — `429 Too Many Requests`: https://datatracker.ietf.org/doc/html/rfc9110
- RFC 9110 §10.2.4 — `Retry-After`: https://datatracker.ietf.org/doc/html/rfc9110#section-10.2.4
- draft-ietf-httpapi-ratelimit-headers: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- Cloudflare sliding window counter: https://blog.cloudflare.com/counting-things-a-lot-of-different-things/
- Stripe rate limits: https://stripe.com/docs/rate-limits

## Related

- [[In-process Rate Limiting Implementation]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[API Gateway Rate Limiting Implementation]]
- [[Rate Limiting in Node and Typescript Comparison]]
- [[API versioning strategies]]
- [[API deprecation and sunset]]
