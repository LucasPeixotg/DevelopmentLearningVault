---
name: api-idempotency
description: Use this skill when adding a POST/PATCH endpoint with side effects clients might retry — charges, payouts, emails, provisioning, webhooks — and you need retries to be safe. Implements the Idempotency-Key pattern with atomic server-side dedup so a retried request never duplicates the operation.
---

# API idempotency

## When to use this

You're adding a write endpoint whose side effect is expensive to duplicate (charge a card,
send money/email/SMS, provision a resource, fire a webhook) and clients may retry under
network uncertainty. `GET`/`PUT`/`DELETE` are already idempotent by the HTTP spec; `POST`
and non-trivial `PATCH` are not — that's where this pattern applies. Goal: **exactly-once**
semantics = at-least-once retries + server-side deduplication.

## Decision

- Add idempotency to **`POST` and non-idempotent `PATCH`** with real side effects. Don't
  add it to `GET` (pointless) or naturally-idempotent writes.
- **Client generates the key**, one per *logical operation*, reused on every retry.
- **Server stores key → response** and replays the stored response on a duplicate, with an
  **atomic lock** so two concurrent retries can't both execute.
- Use **UUID v4/v7** (v7 is time-ordered, better index locality). TTL records **24h**
  (Stripe convention).

## Implementation

### Client: generate → persist → send (in that order)

```typescript
const key = randomUUID();                              // one per logical operation
await db.savePendingPayment({ key, status: 'pending' }); // persist BEFORE sending,
const res = await chargeWithIdempotencyKey(key, amount); //   so a crash doesn't lose the key
await db.updatePayment({ key, status: 'complete', response: res });
```

Send the same key on every attempt; retry with exponential backoff + jitter. Retry on
network errors, `409` (in-progress — wait), `429` (after `Retry-After`), and `5xx`. **Do
not** retry `400`/`401`/`403`/`404`/`422` — the request itself is wrong.

### Server: atomic dedup middleware (Express + Redis)

The hard parts are **atomic locking** (`SET NX`, never GET-then-SET) and **failure
recovery**. Store a body hash to detect key reuse with a different payload.

```typescript
const TTL = 86_400; // 24h
const hashBody = (b: unknown) => createHash('sha256').update(JSON.stringify(b)).digest('hex');

export async function idempotency(req, res, next) {
  const key = req.headers['idempotency-key'] as string | undefined;
  if (!key) return next();                                  // no key → normal request
  if (key.length > 255) return void res.status(400).json({ /* problem+json */ });

  const storeKey = `idempotency:${key}`;
  const bodyHash = hashBody(req.body);
  const raw = await redis.get(storeKey);

  if (raw) {
    const rec = JSON.parse(raw);
    if (rec.request.bodyHash !== bodyHash || rec.request.path !== req.path)
      return void res.status(422).json({ /* key reused with a different request */ });
    if (!rec.response)
      return void res.status(409).json({ /* request-in-progress: wait and retry */ });
    res.set(rec.response.headers);                          // replay the ORIGINAL response
    return void res.status(rec.response.status).send(rec.response.body);
  }

  // New key — atomic lock: SET NX only succeeds if the key does not exist
  const locked = await redis.set(storeKey, JSON.stringify({
    key, request: { method: req.method, path: req.path, bodyHash }, response: null,
    createdAt: new Date().toISOString() }), 'EX', TTL, 'NX');
  if (!locked) return void res.status(409).json({ /* lost the race → in-progress */ });

  // Capture the response and persist it on finish, so retries replay it
  let status = 500, body = '';
  const json = res.json.bind(res);
  res.json = (b: unknown) => { status = res.statusCode; body = JSON.stringify(b); return json(b); };
  res.on('finish', async () => {
    if (!body) return;
    const stored = JSON.parse((await redis.get(storeKey))!);
    stored.response = { status, headers: { 'content-type': 'application/json' }, body };
    await redis.set(storeKey, JSON.stringify(stored), 'EX', TTL);
  });
  next();
}

// Apply only to non-idempotent routes:
app.post('/v1/charges', idempotency, chargesHandler);
```

### Failure recovery

- **Server crashes mid-process** → record stays `response: null`; retries get `409` until
  it clears. Use a **separate short-lived lock** (`idempotency:lock:<key>`, ~30s TTL) so the
  lock expires and a retry can re-process, while the 24h main record holds the response.
- **Response sent but not received** → retry replays the stored response. *This is the
  happy path the pattern exists for.*
- **Client crashes before receiving** → mitigated by persisting the key before sending
  (above), so on restart it can find the `pending` row and retry with the same key.

## Checklist

- [ ] Key generated client-side, one per logical operation, persisted before first send.
- [ ] Reused on every retry (never a fresh key per attempt).
- [ ] Server uses an **atomic** `SET NX` / `INSERT ... ON CONFLICT DO NOTHING` — never
      check-then-write.
- [ ] Stores request body hash + path; rejects key reuse with a different request (`422`).
- [ ] Concurrent duplicate → `409` (client waits and retries).
- [ ] Replays the **exact** original response (status + body).
- [ ] Records have a TTL (24h); response-store errors are caught + alerted.
- [ ] Applied only to non-idempotent write routes; keys capped at 255 chars (`400` if longer).

## Pitfalls

- **New key per retry** — defeats the whole pattern.
- **Reusing one key across different operations** — collisions and wrong replays.
- **GET-then-SET locking** — race condition; two requests both execute.
- **Not hashing the body** — a client reuses the key with a different amount.
- **No TTL** — the store grows forever.
- **Replaying a different response** — confuses clients.
- **Silent failure storing the response** — record stuck `null`, all retries get `409` forever.
- **Sequential IDs as keys** — guessable; attackers replay other operations. Use UUID v4/v7.

## Reference

Vault notes:
- `API Best Practices/Idempotency/Idempotency.md`
- `API Best Practices/Idempotency/Idempotency-Key header pattern.md`
- `API Best Practices/Idempotency/Idempotency server-side implementation.md`

External: IETF Idempotency-Key draft
(https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) ·
Stripe idempotent requests (https://stripe.com/docs/api/idempotent-requests) ·
Brandur, "Idempotency Keys in Postgres" (https://brandur.org/idempotency-keys)

Related skills: `rest-api-error-design` (the 409/422 problem bodies),
`api-rate-limiting` (the 429 + Retry-After clients also retry on).
