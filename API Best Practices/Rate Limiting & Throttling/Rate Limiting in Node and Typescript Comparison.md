---
tags: [api-design, typescript, nodejs, rate-limiting]
created: 2026-06-03
---
# Rate Limiting in Node and Typescript Comparison

> Summary: A decision guide across the four rate limiting approaches available in a Node.js/TypeScript + Express stack. The approaches are not mutually exclusive — most production systems layer two or three of them. Key dimensions: correctness across servers, operational cost, flexibility for custom logic, and testability.

## At a glance

| | [[In-process Rate Limiting Implementation\|In-process]] | [[Redis-backed sliding window rate limiting Implementation\|Redis + Lua]] | [[Redis Libraries Rate Limiting Implementation\|express-rate-limit]] | [[API Gateway Rate Limiting Implementation\|API Gateway]] |
|---|---|---|---|---|
| **Multi-server correct** | No | Yes | Yes | Yes |
| **External dependency** | None | Redis | Redis | Gateway |
| **Algorithm** | Fixed window | Sliding window | Fixed or sliding | Depends on gateway |
| **Burst handling** | Poor | Good | Moderate | Good (token bucket on AWS/Kong) |
| **Custom logic** | Full control | Full control | Limited | Very limited |
| **Code to maintain** | ~50 lines | ~80 lines | ~30 lines | 0 lines |
| **Protects at** | App layer | App layer | App layer | Gateway layer |
| **Survives restarts** | No | Yes | Yes | Yes |
| **Local dev** | Easy | Needs Redis | Needs Redis | Needs gateway |
| **Recommended for** | Dev / prototype | Multi-server, custom logic | Multi-server, standard cases | Scale / infrastructure |

## Decision guide

> [!tip] Start here
> - **Prototyping or single server** → [[In-process Rate Limiting Implementation]]
> - **Multi-server, standard use case** → [[Redis Libraries Rate Limiting Implementation]]
> - **Multi-server, custom plan-tier logic** → [[Redis-backed sliding window rate limiting Implementation]]
> - **High scale or DDoS protection** → [[API Gateway Rate Limiting Implementation]] + one of the Redis options for business logic
> - **Already using Kong or AWS API Gateway** → [[API Gateway Rate Limiting Implementation]] for infrastructure, [[Redis Libraries Rate Limiting Implementation]] for per-user logic

## When to layer approaches

Most production APIs use two layers simultaneously:

```
Request
   ↓
[API Gateway] — coarse limit, blocks DDoS, protects infrastructure
   ↓
[Express middleware (Redis)] — per-user, per-plan, per-endpoint business logic
   ↓
Handler
```

The gateway handles unauthenticated traffic and infrastructure-level protection. Application-level middleware handles business rules (plan tiers, per-endpoint cost) that require your database or auth context.

> [!warning] Don't use in-process in multi-server production
> With N servers, the effective limit becomes `maxRequests × N`. This isn't a subtle failure — a client with a 100 req/min limit gets 300 req/min with 3 pods.

## Algorithm trade-offs

| Algorithm | Used in | Boundary burst | Memory | Complexity |
|---|---|---|---|---|
| Fixed window | In-process, express-rate-limit default | Yes | Very low | Low |
| Sliding window (counter) | Redis + Lua, rate-limit-redis store | No | Low | Medium |
| Token bucket | AWS API Gateway, Kong | N/A (by design) | Low | Medium |
| Leaky bucket | Nginx | N/A (queues) | Low | Medium |

For most APIs the sliding window counter is the right choice: no boundary burst, low memory, near-exact accuracy.

## Header standards

Both Redis approaches and the gateway are configurable. Prefer the IETF draft headers — set once and move on:

```
# Emit this:
RateLimit-Limit: 1000
RateLimit-Remaining: 943
RateLimit-Reset: 57        ← seconds until reset (unambiguous)

# Not this (legacy, ambiguous Reset semantics):
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 943
X-RateLimit-Reset: 1735693200
```

`express-rate-limit` with `standardHeaders: 'draft-7'` handles this automatically. For the custom Redis implementation, set headers manually as shown in [[Redis-backed sliding window rate limiting Implementation]].

## Library quick reference

| Library | Role | Notes |
|---|---|---|
| `express-rate-limit` | Middleware + algorithm | Active maintenance; use `standardHeaders: 'draft-7'` |
| `rate-limit-redis` | Store for express-rate-limit | Official companion; uses `ioredis` or `node-redis` |
| `ioredis` | Redis client | Preferred for advanced use (Lua scripts, pipelines) |
| `node-redis` | Redis client | Official Redis client; simpler API |
| `mnemonist/lru-map` | Bounded in-memory store | Use with in-process approach to prevent memory leaks |

## Related

- [[Rate limiting & throttling]]
- [[In-process Rate Limiting Implementation]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[API Gateway Rate Limiting Implementation]]
- [[Authentication strategies compared]]
