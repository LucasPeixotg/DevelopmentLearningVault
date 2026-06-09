---
tags: [api-design, reliability, http, typescript]
created: 2026-06-03
---

# Idempotency-Key header pattern

> Summary: The client generates a unique key per *logical operation*, sends it as an `Idempotency-Key` request header, and reuses that exact key on every retry. The server uses the key to deduplicate requests and replay the original response. Pioneered by Stripe; now an IETF draft. Transforms non-idempotent `POST` operations into safely retryable ones.

## The flow

```
Client                              Server
  │  POST /v1/charges               │
  │  Idempotency-Key: key-abc       │
  │ ───────────────────────────────►│
  │                                 │ 1. key-abc not seen before
  │                                 │ 2. Process request (charge card)
  │                                 │ 3. Store key-abc → response
  │  201 Created + charge object    │
  │ ◄───────────────────────────────│
  │                                 │
  │  (network blip — client retries)│
  │  POST /v1/charges               │
  │  Idempotency-Key: key-abc       │
  │ ───────────────────────────────►│
  │                                 │ 1. key-abc found in store
  │                                 │ 2. Return stored response — no charge
  │  201 Created + same object      │
  │ ◄───────────────────────────────│
```

The second response is identical to the first. No duplicate charge.

## The key

- **Generated client-side** — the client owns the key, not the server.
- **One key per logical operation** — not per request. The same logical operation reuses the same key across retries.
- **UUID v4 or UUID v7** — UUID v7 is time-ordered and increasingly preferred (better database index locality).
- **Persisted before the first request** — if the client crashes after sending but before receiving the response, it needs the key to retry correctly (see [[Idempotency server-side implementation]]).

```typescript
import { randomUUID } from 'crypto';

// Generate ONCE per logical operation, before the first attempt
const idempotencyKey = randomUUID();

// Persist it — so a client crash doesn't lose the key
await db.savePendingOperation({ key: idempotencyKey, status: 'pending' });
```

## Client-side retry with exponential backoff

```typescript
async function chargeWithRetry(
  amount: number,
  currency: string,
  idempotencyKey: string,     // passed in — generated and persisted by caller
  maxAttempts = 4
): Promise<ChargeResponse> {

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      const res = await fetch('https://api.example.com/v1/charges', {
        method: 'POST',
        headers: {
          'Authorization':   `Bearer ${process.env.API_KEY}`,
          'Idempotency-Key': idempotencyKey,   // same key every attempt
          'Content-Type':    'application/json',
        },
        body: JSON.stringify({ amount, currency }),
      });

      // Success — return
      if (res.ok) return res.json();

      // 409 = request in progress (concurrent duplicate) — wait and retry
      if (res.status === 409) {
        await sleep(backoffMs(attempt));
        continue;
      }

      // 4xx (except 409, 429) = client error — don't retry
      if (res.status >= 400 && res.status < 500) {
        throw new ApiError(await res.json());
      }

      // 5xx = server error — retry
    } catch (err) {
      if (err instanceof ApiError) throw err;     // non-retryable
      if (attempt === maxAttempts - 1) throw err; // out of attempts
    }

    await sleep(backoffMs(attempt));
  }

  throw new Error('Max retry attempts exceeded');
}

function backoffMs(attempt: number): number {
  // Exponential backoff with jitter: 100ms, 200ms, 400ms, 800ms...
  return Math.min(100 * 2 ** attempt, 10_000)
    + Math.random() * 100;  // jitter prevents thundering herd
}
```

## What statuses are retryable?

| Status | Retry? | Reason |
|---|---|---|
| Network error / timeout | Yes | Server may not have received request |
| `409 Conflict` | Yes (with delay) | Concurrent duplicate — wait for original to finish |
| `429 Too Many Requests` | Yes (after `Retry-After`) | Rate limited |
| `500`, `502`, `503`, `504` | Yes | Server error |
| `400`, `401`, `403`, `404`, `422` | No | Client error — retrying won't help |
| `200`, `201`, `204` | N/A | Success |

> [!warning] Don't retry on `400`/`422`
> A validation error won't resolve itself on retry. These statuses mean the request itself is wrong — fix it, don't retry it.

## Key rules

> [!warning] Common client-side mistakes
> - **Generating a new key per retry.** Defeats the pattern entirely — each retry looks like a fresh operation.
> - **Reusing a key across different operations.** One key = one logical operation. Don't reuse "last month's payment key" for this month's charge.
> - **Not persisting the key before the first request.** If the client crashes after sending, it needs the key to retry. Generate → persist → send, in that order.
> - **Using sequential IDs as keys.** Guessable keys let attackers replay other operations. Use UUID v4/v7.

## Header format

Per the IETF draft, the header name is `Idempotency-Key` (case-insensitive by HTTP spec) and the value should be a unique string of up to 255 characters:

```http
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

The server should reject keys longer than 255 characters with `400 Bad Request`.

## References

- draft-ietf-httpapi-idempotency-key-header: https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- Stripe idempotent requests: https://stripe.com/docs/api/idempotent-requests
- UUID v7 spec: https://www.ietf.org/rfc/rfc9562.html#section-5.7

## Related

- [[Idempotency]]
- [[Idempotency server-side implementation]]
- [[Rate limiting & throttling]]
- [[Error response formats]]
