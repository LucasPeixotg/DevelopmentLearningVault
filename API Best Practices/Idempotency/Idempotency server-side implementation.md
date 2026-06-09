---
tags: [api-design, reliability, redis, typescript, implementation]
created: 2026-06-03
---

# Idempotency server-side implementation

> Summary: The server stores each `Idempotency-Key` alongside the original response. On retry, it looks up the key and replays the stored response without re-executing the operation. The two hard problems: **atomic locking** (two simultaneous requests with the same key must not both execute), and **failure recovery** (what happens when the server crashes before storing the response). Implemented here with Redis `SET NX` for atomic locking.

## What to store

```typescript
interface IdempotencyRecord {
  key: string;
  request: {
    method:   string;
    path:     string;
    bodyHash: string;    // SHA-256 of request body — detects key reuse with different body
  };
  response: {
    status:  number;
    headers: Record<string, string>;
    body:    string;     // serialised JSON
  } | null;              // null = request is in flight
  createdAt: string;     // ISO 8601
}
```

TTL: **24 hours** (Stripe's convention). Adjust based on how long clients realistically retry.

## The locking problem

Two clients sending the same key simultaneously (a client that retried before the first attempt finished):

```
Client A ──► POST /charges (key-abc) ──► server starts processing
Client B ──► POST /charges (key-abc) ──► must not also process
```

The solution is an **atomic SET NX** — Redis's "set if not exists." Only one request wins; the other gets `409 Conflict` and retries after a delay.

> [!warning] Never use GET then SET for locking
> Two servers can both GET, find nothing, then both SET — both process the request and you have a duplicate. The check and the write must be a single atomic operation.

## Implementation (Express + Redis)

```typescript
import { createHash }   from 'crypto';
import { Redis }        from 'ioredis';
import { Request, Response, NextFunction } from 'express';

const redis      = new Redis(process.env.REDIS_URL!);
const TTL        = 86_400;   // 24 hours in seconds

function hashBody(body: unknown): string {
  return createHash('sha256').update(JSON.stringify(body)).digest('hex');
}

export async function idempotencyMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> {
  const key = req.headers['idempotency-key'] as string | undefined;
  if (!key) return next();   // no key → normal request

  if (key.length > 255) {
    return void res.status(400).json({
      type:   'https://api.example.com/errors/invalid-idempotency-key',
      title:  'Idempotency-Key must be 255 characters or fewer.',
      status: 400,
    });
  }

  const storeKey = `idempotency:${key}`;
  const bodyHash = hashBody(req.body);

  // ── Check for existing record ─────────────────────────────────────────
  const raw = await redis.get(storeKey);

  if (raw) {
    const record: IdempotencyRecord = JSON.parse(raw);

    // Key reused with different body or path → reject
    if (record.request.bodyHash !== bodyHash ||
        record.request.path    !== req.path) {
      return void res.status(422).json({
        type:   'https://api.example.com/errors/idempotency-key-reused',
        title:  'Idempotency-Key reused with a different request.',
        status: 422,
      });
    }

    // Request still in flight (concurrent duplicate)
    if (!record.response) {
      return void res.status(409).json({
        type:   'https://api.example.com/errors/request-in-progress',
        title:  'A request with this Idempotency-Key is already in progress.',
        status: 409,
        detail: 'Wait briefly and retry — the original request is still processing.',
      });
    }

    // Replay stored response exactly
    res.set(record.response.headers);
    return void res.status(record.response.status).send(record.response.body);
  }

  // ── New key — atomic lock ─────────────────────────────────────────────
  const initialRecord: IdempotencyRecord = {
    key,
    request:   { method: req.method, path: req.path, bodyHash },
    response:  null,
    createdAt: new Date().toISOString(),
  };

  // SET NX — only succeeds if key does not exist
  const locked = await redis.set(
    storeKey,
    JSON.stringify(initialRecord),
    'EX', TTL,
    'NX'
  );

  if (!locked) {
    // Lost the race to another server instance
    return void res.status(409).json({
      type:   'https://api.example.com/errors/request-in-progress',
      title:  'A request with this Idempotency-Key is already in progress.',
      status: 409,
    });
  }

  // ── Intercept response to store it ───────────────────────────────────
  let capturedStatus = 500;
  let capturedBody   = '';

  const originalJson = res.json.bind(res);
  res.json = function (body: unknown) {
    capturedStatus = res.statusCode;
    capturedBody   = JSON.stringify(body);
    return originalJson(body);
  };

  res.on('finish', async () => {
    if (!capturedBody) return;  // handler didn't call res.json — skip
    const stored: IdempotencyRecord = JSON.parse((await redis.get(storeKey))!);
    stored.response = {
      status:  capturedStatus,
      headers: { 'content-type': 'application/json' },
      body:    capturedBody,
    };
    await redis.set(storeKey, JSON.stringify(stored), 'EX', TTL);
  });

  next();
}
```

```typescript
// app.ts — apply only to non-idempotent routes
app.post('/v1/charges', idempotencyMiddleware, chargesHandler);
app.post('/v1/refunds', idempotencyMiddleware, refundsHandler);
// GET, PUT, DELETE don't need it — they're naturally idempotent
```

## Failure recovery scenarios

### Scenario 1: Server crashes mid-processing

Record in Redis has `response: null`. The lock hasn't expired yet. On retry, client gets `409 Conflict`.

**Fix — separate short-lived lock:**
Use a secondary key `idempotency:lock:key-abc` with a short TTL (e.g. 30s). The main record (24h TTL) stores the response once complete. If the server crashes, the lock expires in 30s and the next retry can re-process:

```typescript
// Acquire short-lived lock separately
const lockKey = `idempotency:lock:${key}`;
const lockAcquired = await redis.set(lockKey, '1', 'EX', 30, 'NX');
if (!lockAcquired) {
  return void res.status(409).json({ ... });
}
// On finish: store response in main key, lock expires naturally
```

### Scenario 2: Response sent but not received

Server stored the response and sent `201 Created`. Network dropped it. Client retries with the same key. Server replays stored response. **This is the happy path — exactly what the pattern is for.**

### Scenario 3: Client crashes before receiving

Client never recorded the key it sent. On restart, it generates a new UUID. The server creates a new charge. **Potential duplicate.**

Prevention: persist the key *before* sending:

```typescript
// Right order: generate → persist → send
const key = randomUUID();
await db.savePendingPayment({ key, status: 'pending' });        // persist first
const response = await chargeWithIdempotencyKey(key, amount);   // then send
await db.updatePayment({ key, status: 'complete', response });
```

If the client crashes between steps, it can query its own DB for any `pending` payment, find the key, and retry with it.

## Common mistakes

> [!warning] Anti-patterns
> - **Not storing the request body hash.** A client can reuse the same key with a different amount. Always validate body + path match the original request.
> - **No TTL on records.** The store grows forever. 24 hours is the Stripe convention.
> - **Using a database without atomic writes.** A non-atomic check-then-insert has a race condition under concurrent retries. Use `INSERT ... ON CONFLICT DO NOTHING` or Redis `SET NX`.
> - **Not replaying the exact original response.** If the replay returns a different body, clients get confused or make wrong decisions.
> - **Applying to GET requests.** Unnecessary — GETs are naturally idempotent.
> - **Silent failure when storing the response.** If the `finish` handler throws, the record stays `response: null` and all future retries get `409`. Add error handling and alerting.
> - **Not handling `409` on the client.** Clients should treat `409` from an in-progress idempotent request as "wait and retry," not a permanent failure.

## References

- Stripe idempotency implementation: https://stripe.com/docs/api/idempotent-requests
- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres": https://brandur.org/idempotency-keys
- Redis SET NX: https://redis.io/docs/latest/commands/set/
- draft-ietf-httpapi-idempotency-key-header: https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/

## Related

- [[Idempotency]]
- [[Idempotency-Key header pattern]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[Error response formats]]
