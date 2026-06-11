---
name: api-pagination
description: Use this skill when an API endpoint returns a list or collection — choosing offset vs page vs cursor pagination, designing the response envelope, writing the SQL, or securing cursors. Produces a scalable, consistent pagination contract.
---

# API pagination

## When to use this

Any endpoint that returns a collection (`GET /orders`, `/users`, a feed). Pagination is
where APIs degrade silently under load — page 1 is fast, page 5000 may not be — so pick
the algorithm deliberately and lock the envelope shape across all list endpoints.

## Decision (pick an algorithm)

| | Offset/limit | Page-based | Cursor-based |
|---|---|---|---|
| Random access (jump to page N) | Yes | Yes | **No** |
| Cheap total count | Yes | Yes | **No** (full scan) |
| Performance at depth | Degrades | Degrades | **Constant** |
| Consistent under inserts/deletes | No | No | **Yes** |
| Complexity | Low | Low | Medium |

- **Default to cursor-based** for large, growing, or frequently-mutating data, feeds, and
  public APIs (Stripe, GitHub, Slack). Constant-time at any depth, stable under mutation.
- **Offset/limit** only for admin UIs / small datasets where you need random page access
  and a cheap `totalCount`.
- **Page-based** is offset with different math (`page`/`perPage`) — same trade-offs.

Always have a **default limit (20)** and a **hard maximum (100)**; reject `?limit=1000000`
with `400`. Use one envelope shape everywhere.

## Implementation (cursor-based, Express + PostgreSQL)

### Envelope

```json
{ "data": [...], "pagination": { "nextCursor": "cursor_...", "hasMore": true } }
```

`nextCursor` is `null` on the last page; `hasMore` mirrors `nextCursor !== null`.

### Cursor encode/decode

Encode the **immutable** sort-column values of the last row (`created_at` + unique `id`
tiebreaker). Always Base64url at minimum so clients can't build cursors by hand:

```typescript
const encodeCursor = (r: { created_at: Date; id: string }) =>
  Buffer.from(JSON.stringify({ created_at: r.created_at.toISOString(), id: r.id })).toString('base64url');

const decodeCursor = (c: string) => JSON.parse(Buffer.from(c, 'base64url').toString());
```

### Keyset query (constant-time at any depth)

```sql
-- needs a composite index matching the ORDER BY, in order:
CREATE INDEX idx_orders_pagination ON orders (created_at DESC, id DESC);

SELECT id, amount, created_at FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)   -- row-value comparison; seeks via index
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

For DBs without row-value comparison (older MySQL/SQLite), expand:
`WHERE created_at < :c OR (created_at = :c AND id < :id)`. Verify with `EXPLAIN` that it's
an Index Scan, not a Seq Scan — a keyset query without the index is no faster than offset.

### Handler

```typescript
app.get('/v1/orders', async (req, res) => {
  const limit = Math.min(Number(req.query.limit ?? 20), 100);
  let where = ''; const params: unknown[] = [limit];
  if (req.query.after) {
    try {
      const { created_at, id } = decodeCursor(req.query.after as string);
      params.push(created_at, id);
      where = 'WHERE (created_at, id) < ($2, $3)';
    } catch {
      return res.status(400).contentType('application/problem+json').json({
        type: 'https://api.example.com/errors/invalid-cursor',
        title: 'Invalid or expired pagination cursor.', status: 400,
        detail: 'Restart pagination from the first page.' });
    }
  }
  const rows = await db.query(
    `SELECT id, amount, created_at FROM orders ${where} ORDER BY created_at DESC, id DESC LIMIT $1`, params);
  const nextCursor = rows.length === limit ? encodeCursor(rows[rows.length - 1]) : null;
  res.json({ data: rows, pagination: { nextCursor, hasMore: nextCursor !== null } });
});
```

## Cursor security (encode → sign → encrypt)

| Level | Method | Client can read / tamper | Use when |
|---|---|---|---|
| 2 | Base64url | reads + tampers easily | internal/dev only |
| 3 | HMAC-SHA256 sign (`data.sig`, `timingSafeEqual`) | reads, can't tamper | public API, non-sensitive |
| 4 | AES-256-GCM encrypt (12-byte IV + 16-byte tag + ciphertext) | neither | sensitive values, multi-tenant |

Most production APIs land at level 3 or 4. With level 4 you can also bind `user_id`
(reject another user's cursor), add `exp` (expiry), and `version` (bump to invalidate all
cursors). Generate keys with `openssl rand -hex 32`; store in a secret manager; rotate via
the `version` field. **Return the same error for every cursor failure** (tampered /
expired / wrong user) — differences leak information.

### `totalCount`

Cheap for offset (`COUNT(*)`), expensive for cursor (full scan per page). Most cursor APIs
omit it (Stripe, GitHub); if needed, expose a separate `GET /orders/count` or return it
only on the first page. Committing to `totalCount` on every cursor response is a trap.

## Checklist

- [ ] Default limit (20) and enforced maximum (100); `?limit` over max → `400`.
- [ ] One envelope shape across all list endpoints.
- [ ] Stable, deterministic `ORDER BY` including a unique tiebreaker (`id`).
- [ ] Cursor on **immutable** columns only (`created_at` + `id`), never `updated_at`.
- [ ] Composite index matching the ORDER BY; verified with `EXPLAIN`.
- [ ] Cursors at least Base64url-encoded; signed/encrypted if public or sensitive.
- [ ] Same generic error for all cursor failures.
- [ ] Explicit empty-last-page (`hasMore: false`, `nextCursor: null`).

## Pitfalls

- **No default/max limit** — `GET /orders` returns everything.
- **Cursoring on mutable columns** (`updated_at`) — records jump position.
- **Missing index** — keyset query degrades to a scan, no better than offset.
- **`totalCount` on every cursor page** — full scan per request.
- **No tiebreaker** — duplicate `created_at` ⇒ non-deterministic order, unreliable cursors.
- **Unencoded cursors** — clients construct them manually and you can never change internals.
- **Inconsistent envelopes across endpoints** — clients can't write generic handling.

## Reference

Vault notes:
- `API Best Practices/Pagination/Pagination Introduction.md`
- `API Best Practices/Pagination/Cursor-based Pagination.md`
- `API Best Practices/Pagination/Offset-limit Pagination.md`
- `API Best Practices/Pagination/Page-based Pagination.md`
- `API Best Practices/Pagination/Cursor Encryption.md`

External: "We need to talk about offset pagination" (https://use-the-index-luke.com/no-offset) ·
Stripe pagination (https://stripe.com/docs/api/pagination) ·
Relay cursor connections (https://relay.dev/graphql/connections.htm)

Related skills: `rest-api-error-design` (the invalid-cursor error body),
`api-authentication` (binding `user_id` into cursors).
