---
tags: [api-design, reliability, distributed-systems, best-practices]
created: 2026-06-03
---
# Idempotency

> Summary: An operation is idempotent if performing it multiple times produces the same result as performing it once. Idempotency is how you make retries safe — essential when networks drop responses and clients can't tell if a request was processed. `GET`, `PUT`, and `DELETE` are naturally idempotent by the HTTP spec; `POST` and `PATCH` are not by default, which is why the [[Idempotency-Key header pattern]] exists.

## The core problem

Networks fail. Clients time out. Servers crash mid-request. When this happens, the client faces one question: **did the server process my request or not?**

- For `GET /orders` — doesn't matter. Fetching twice is harmless.
- For `POST /charges` — matters enormously. Retrying could charge the customer twice.

Without idempotency, clients must choose between potentially duplicating operations (retry) or potentially losing them (don't retry). Idempotency eliminates that dilemma.

## Formal definition

```
f(f(x)) = f(x)
```

An operation is idempotent if its **side effects on the server** are the same whether it runs once or many times. The response code may differ (a second `DELETE` can return `404`) — what matters is the server state.

## HTTP methods and idempotency

| Method | Idempotent | Safe | Notes |
|---|---|---|---|
| `GET` | Yes | Yes | Pure read — no side effects. |
| `HEAD` | Yes | Yes | Same as GET, no body. |
| `OPTIONS` | Yes | Yes | Metadata only. |
| `PUT` | Yes | No | Replaces entire resource — same payload, same result. |
| `DELETE` | Yes | No | Resource is gone after first call; subsequent calls are no-ops. |
| `POST` | **No** | No | Creates new resources by default. The main problem case. |
| `PATCH` | **No** | No | Depends on the operation — setting a value is idempotent, incrementing is not. |

> [!warning] `DELETE` returning `404` on retry
> `DELETE` is idempotent — the server state is the same after any number of calls. But many implementations return `404` on the second call. This is spec-compliant. Client code should treat `404` on a `DELETE` retry as success, not an error.

> [!tip] `PATCH` idempotency depends on the operation
> `PATCH /orders/42 { "status": "shipped" }` — idempotent (sets a value).
> `PATCH /orders/42 { "quantity": { "increment": 1 } }` — not idempotent (relative change).
> Whether PATCH is idempotent is a design decision, not a guarantee.

## At-most-once, at-least-once, exactly-once

These distributed systems terms describe retry behaviour:

| Delivery | Behaviour | Risk |
|---|---|---|
| **At-most-once** | Send once, never retry | Operations may be lost |
| **At-least-once** | Retry until acknowledged | Duplicates without deduplication |
| **Exactly-once** | At-least-once + server deduplication | What `Idempotency-Key` achieves |

The [[Idempotency-Key header pattern]] achieves exactly-once semantics by combining at-least-once retries with server-side deduplication. The client retries freely; the server detects duplicates and replays the original response.

## When idempotency matters most

- **Financial operations** — charges, refunds, payouts. Duplicates are catastrophic.
- **State transitions** — order status changes, user provisioning. Applying twice must be safe.
- **External side effects** — sending emails, SMS, webhooks. Retrying without deduplication sends duplicates.
- **Any `POST` or non-idempotent `PATCH` that clients might retry** under network uncertainty.

## References

- RFC 9110 §9.2.2 — idempotent methods: https://datatracker.ietf.org/doc/html/rfc9110#section-9.2.2
- Stripe idempotency docs: https://stripe.com/docs/api/idempotent-requests
- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres": https://brandur.org/idempotency-keys

## Related

- [[Idempotency-Key header pattern]]
- [[Idempotency server-side implementation]]
- [[API Versioning Strategies]]
- [[Rate limiting & throttling]]
- [[Error response formats]]
