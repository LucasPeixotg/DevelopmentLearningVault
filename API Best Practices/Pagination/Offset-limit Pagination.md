---
tags: [api-design, pagination, database, sql]
created: 2026-06-03
---

# Offset/limit Pagination

> Summary: The simplest pagination approach — the client specifies how many records to skip (`offset`) and how many to return (`limit`). Trivial to implement and supports random access and `totalCount`. Breaks down in two ways: **performance degrades at large offsets** (the database scans and discards every skipped row), and **mutations during pagination cause duplicates or skips**. Use it for small datasets, admin UIs, and slowly mutating data. Use [[Cursor-based Pagination]] for large or frequently mutating data.

## Wire format

```http
GET /v1/orders?offset=0&limit=20    ← first page
GET /v1/orders?offset=20&limit=20   ← second page
GET /v1/orders?offset=40&limit=20   ← third page
```

## Response envelope

```json
{
  "data": [...],
  "pagination": {
    "offset": 20,
    "limit":  20,
    "total":  1543,
    "hasMore": true
  }
}
```

## How it works (SQL)

```sql
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

The database scans the first 40 rows, discards them, then returns the next 20. At `OFFSET 40`, that's cheap. At `OFFSET 10000000`, the database has scanned 10 million rows just to throw them away.

## Performance problem

`OFFSET N` forces a full scan of N rows regardless of indexes. The deeper you paginate, the slower each page gets — linearly:

```
OFFSET 0        → fast   (no rows discarded)
OFFSET 1000     → slow   (1,000 rows discarded)
OFFSET 100000   → slower (100,000 rows discarded)
OFFSET 10000000 → very slow (10M rows discarded)
```

This is why offset pagination is unsuitable for large datasets. Use [[Cursor-based Pagination]] (keyset queries) for constant-time performance at any depth.

## Consistency problem

If records are inserted or deleted between pages, the offset shifts and you get duplicates or skips:

```
Page 1 fetched (OFFSET 0):  rows A B C ... T
New row X inserted at position 1
Page 2 fetched (OFFSET 20): row T appears again ← duplicate
```

Conversely, if a row is deleted, a record gets skipped entirely. For slowly mutating data (product catalogues, static reports), this is acceptable. For frequently mutating data (feeds, transactions), it's a real problem.

## TypeScript implementation (Express + PostgreSQL)

```typescript
app.get('/v1/orders', async (req, res) => {
  const limit  = Math.min(Number(req.query.limit  ?? 20), 100); // cap at 100
  const offset = Math.max(Number(req.query.offset ?? 0),  0);

  if (isNaN(limit) || isNaN(offset)) {
    return res.status(400).json({ title: 'Invalid pagination parameters', status: 400 });
  }

  const [rows, [{ count }]] = await Promise.all([
    db.query(
      'SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT $1 OFFSET $2',
      [limit, offset]
    ),
    db.query('SELECT COUNT(*)::int AS count FROM orders'),
  ]);

  res.json({
    data: rows,
    pagination: {
      offset,
      limit,
      total:   count,
      hasMore: offset + rows.length < count,
    },
  });
});
```

## When to use

- Admin UIs where users jump to arbitrary pages ("page 47 of 312")
- Slowly mutating data where occasional duplicates/skips are acceptable
- Small datasets where depth performance doesn't matter
- When `totalCount` is genuinely needed by the UI and cost is acceptable

## When not to use

- Large tables (millions of rows) — use [[Cursor-based Pagination]]
- Frequently mutating data — use [[Cursor-based Pagination]]
- Infinite scroll UIs — cursor-based is simpler and more correct

## Common mistakes

> [!warning] Anti-patterns
> - **No default limit.** Without a cap, `GET /orders` returns every row.
> - **Not capping `limit`.** `?limit=1000000` should return `400`, not 1M rows.
> - **No tiebreaker in ORDER BY.** `ORDER BY created_at DESC` is non-deterministic when two rows share a timestamp. Always add `id` as a tiebreaker.
> - **Running `COUNT(*)` on every page.** Count once on the first page and let the client cache it, or skip it entirely for large tables.
> - **Negative `offset`.** Validate that `offset >= 0`.

## References

- PostgreSQL LIMIT/OFFSET docs: https://www.postgresql.org/docs/current/queries-limit.html
- "We need to talk about offset pagination": https://use-the-index-luke.com/no-offset

## Related

- [[Pagination Introduction]]
- [[Page-based Pagination]]
- [[Cursor-based Pagination]]
- [[Cursor Encryption]]
