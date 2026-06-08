---
tags: [api-design, pagination, database]
created: 2026-06-03
---

# Page-based Pagination

> Summary: Syntactic sugar over [[Offset-limit Pagination]] — `page=3&per_page=20` is just `offset=40&limit=20` with different math. More human-readable, but shares all the same problems: performance degrades at depth, mutations cause duplicates or skips, and changing `per_page` between requests breaks page numbers. Use it for the same cases as offset/limit; consider [[Cursor-based Pagination]] for large or frequently mutating data.

## Wire format

```http
GET /v1/orders?page=1&per_page=20    ← first page
GET /v1/orders?page=2&per_page=20    ← second page
GET /v1/orders?page=3&per_page=20    ← third page
```

## Response envelope

```json
{
  "data": [...],
  "pagination": {
    "page":       2,
    "per_page":   20,
    "total_pages": 78,
    "total":      1543,
    "hasMore":    true
  }
}
```

## How it maps to offset/limit

```typescript
const offset = (page - 1) * per_page;
// page=3, per_page=20 → offset=40, limit=20
```

```sql
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 40;
```

It's the same query. Same index behaviour, same scan cost. Page-based pagination is offset pagination with a friendlier URL.

## The `per_page` change trap

If the client changes `per_page` between requests, page numbers become meaningless:

```
First request:  page=3, per_page=20  → rows 41–60
Second request: page=3, per_page=50  → rows 101–150  ← completely different rows
```

Clients that cache `totalPages` and allow users to pick a page number must use a consistent `per_page`. If your API allows the client to change page size mid-session, page numbers are no longer a reliable position indicator.

## TypeScript implementation (Express + PostgreSQL)

```typescript
app.get('/v1/orders', async (req, res) => {
  const perPage = Math.min(Number(req.query.per_page ?? 20), 100);
  const page    = Math.max(Number(req.query.page    ?? 1),  1);

  if (isNaN(perPage) || isNaN(page)) {
    return res.status(400).json({ title: 'Invalid pagination parameters', status: 400 });
  }

  const offset = (page - 1) * perPage;

  const [rows, [{ count }]] = await Promise.all([
    db.query(
      'SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT $1 OFFSET $2',
      [perPage, offset]
    ),
    db.query('SELECT COUNT(*)::int AS count FROM orders'),
  ]);

  const totalPages = Math.ceil(count / perPage);

  res.json({
    data: rows,
    pagination: {
      page,
      per_page:    perPage,
      total_pages: totalPages,
      total:       count,
      hasMore:     page < totalPages,
    },
  });
});
```

## Compared to offset/limit

| | Page-based | Offset/limit |
|---|---|---|
| Human-readable | Yes (`page=3`) | Less so (`offset=40`) |
| Maps to SQL | `(page-1) × per_page` | Directly |
| Total count | Enables `totalPages` | Enables total items |
| Performance at depth | Same — degrades | Same — degrades |
| Consistency under mutations | Same — unreliable | Same — unreliable |
| Per-page change safety | No | N/A |

## When to use

- Same cases as [[Offset-limit Pagination]]: admin UIs, slowly changing data, small datasets.
- When your UI shows "Page 3 of 78" and page-jumping matters to users.
- When `per_page` is fixed (not user-configurable) so page numbers stay meaningful.

## When not to use

- Large tables — performance degrades with depth.
- Frequently mutating data — mutations cause duplicates/skips.
- When `per_page` is variable — page numbers become meaningless.
- Infinite scroll — use [[Cursor-based Pagination]].

## Common mistakes

> [!warning] Anti-patterns
> - **`page=0` instead of `page=1`.** Decide on 0-indexed or 1-indexed and document it. Most human-facing APIs use 1.
> - **Not capping `per_page`.** Always enforce a maximum.
> - **Allowing variable `per_page` while exposing `total_pages`.** `total_pages` is only meaningful for a fixed page size.
> - **No tiebreaker in ORDER BY.** Same issue as offset/limit — add `id` alongside any non-unique sort column.

## References

- PostgreSQL LIMIT/OFFSET: https://www.postgresql.org/docs/current/queries-limit.html
- GitHub's pagination (uses both page-based and cursor): https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api

## Related

- [[Pagination Introduction]]
- [[Offset-limit Pagination]]
- [[Cursor-based Pagination]]
- [[Cursor Encryption]]
