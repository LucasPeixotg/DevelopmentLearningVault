---
tags: [api-design, pagination, database, sql, best-practices]
created: 2026-06-03
---
# Cursor-based Pagination

> Summary: Instead of "skip N rows," the client says "give me records after this specific position." The cursor encodes the sort column values of the last seen record; the server translates it into a keyset SQL query that seeks directly to that position — **constant-time performance at any depth**. Stable under mutations (no duplicates or skips). The trade-off: no random access, no cheap `totalCount`, forward-only by default. Used by Stripe, GitHub, Slack, and Facebook for their primary list endpoints.

## Wire format

```http
GET /v1/orders                         ← first page (no cursor)
GET /v1/orders?after=cursor_Y3JlYXRl   ← second page
GET /v1/orders?after=cursor_aGVsbG8x  ← third page
```

## Response envelope

```json
{
  "data": [...],
  "pagination": {
    "nextCursor":     "cursor_Y3JlYXRl",
    "previousCursor": null,
    "hasMore":        true
  }
}
```

- `nextCursor` — pass as `?after=` to get the next page. `null` on the last page.
- `hasMore` — explicit boolean convenience. Equivalent to `nextCursor !== null`.
- `previousCursor` — for bidirectional navigation. Optional; adds implementation complexity.

## How the cursor works

The cursor encodes the ordering column values of the last record on the page. Always Base64-encode (or encrypt — see [[Cursor Encryption]]) to keep clients from building on the internals.

```typescript
// Encode — called when building the response
function encodeCursor(record: { created_at: Date; id: string }): string {
  return Buffer.from(JSON.stringify({
    created_at: record.created_at.toISOString(),
    id:         record.id,
  })).toString('base64url');
}

// Decode — called when handling the next request
function decodeCursor(cursor: string): { created_at: string; id: string } {
  try {
    return JSON.parse(Buffer.from(cursor, 'base64url').toString());
  } catch {
    throw new Error('Invalid cursor');
  }
}
```

## The keyset SQL query

The cursor translates to a `WHERE` clause that seeks past the last seen record using the index:

```sql
-- First page: no cursor
SELECT id, amount, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Subsequent pages: cursor decoded to (created_at, id)
SELECT id, amount, created_at
FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The composite `WHERE (a, b) < (:a, :b)` is a **row value comparison** — the database uses the index on `(created_at, id)` and seeks directly to the position. **Page 1 and page 500,000 cost the same.**

For databases that don't support row value comparisons (older MySQL, SQLite), expand it:

```sql
WHERE (created_at < :last_created_at)
   OR (created_at = :last_created_at AND id < :last_id)
```

## Index requirement

Without the right index, a keyset query is no faster than offset. Always create a composite index matching the ORDER BY columns, in order:

```sql
CREATE INDEX idx_orders_pagination ON orders (created_at DESC, id DESC);
```

Verify the query uses it with `EXPLAIN`:

```sql
EXPLAIN SELECT ... FROM orders
WHERE (created_at, id) < ('2024-01-15', 'ord_abc')
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Should show "Index Scan" not "Seq Scan"
```

## TypeScript implementation (Express + PostgreSQL)

```typescript
app.get('/v1/orders', async (req, res) => {
  const limit = Math.min(Number(req.query.limit ?? 20), 100);
  let cursorClause = '';
  const params: unknown[] = [limit];

  if (req.query.after) {
    try {
      const { created_at, id } = decodeCursor(req.query.after as string);
      params.push(created_at, id);
      cursorClause = `WHERE (created_at, id) < ($2, $3)`;
    } catch {
      return res.status(400).json({
        type:   'https://api.example.com/errors/invalid-cursor',
        title:  'Invalid or expired pagination cursor.',
        status: 400,
        detail: 'Restart pagination from the first page.',
      });
    }
  }

  const rows = await db.query(
    `SELECT id, amount, created_at FROM orders
     ${cursorClause}
     ORDER BY created_at DESC, id DESC
     LIMIT $1`,
    params
  );

  const lastRow = rows[rows.length - 1];
  const nextCursor = rows.length === limit
    ? encodeCursor(lastRow)
    : null;

  res.json({
    data: rows,
    pagination: {
      nextCursor,
      hasMore: nextCursor !== null,
    },
  });
});
```

## Consistency under mutations

Cursor pagination anchors to a record's values, not a position number. Insertions and deletions outside the current window don't affect the next page:

```
Page 1 fetched: rows ending at record T (cursor = T's values)
New row X inserted at the top of the feed
Page 2 fetched with cursor T: seeks to T, returns rows after T
→ No duplicates, no skips
```

> [!warning] Mutations *within* the window
> If the record the cursor points to is deleted between requests, the keyset query still works — it just seeks to the next record with values less than the cursor. However, if the sort columns of that record are updated (e.g. `updated_at`), the cursor now points to a different position. Cursor only on **immutable columns** — `created_at` + `id` is the standard choice.

## Trade-offs

**Pros:**
- Constant-time performance at any depth.
- Stable under insertions and deletions.
- Naturally maps to infinite scroll UIs.

**Cons:**
- **No random access.** Cannot jump to "page 47." Forward-only by default.
- **No cheap `totalCount`.** Requires a full table scan. Most cursor-based APIs omit it.
- **Cursor must be stable.** Changing the sort order invalidates all existing cursors.
- **Bidirectional navigation is complex.** Requires both `after` and `before` cursors and careful SQL.
- **More complex to implement** than offset, client and server side.

## When to use

- Large datasets (millions of rows)
- Frequently mutating data (feeds, transactions, activity streams)
- Infinite scroll UIs
- Public APIs where clients iterate through large result sets

## Common mistakes

> [!warning] Anti-patterns
> - **Exposing unencoded cursors.** Clients will construct them manually. Always Base64-encode at minimum; encrypt for sensitive data (see [[Cursor Encryption]]).
> - **Cursoring on mutable columns.** `updated_at` as the sort key means records jump position. Use `created_at` + `id`.
> - **Missing index on sort columns.** Keyset queries without the right index are no faster than offset.
> - **Returning `totalCount` on every page.** Forces a full table scan per request.
> - **No tiebreaker column.** Two records with the same `created_at` produce non-deterministic ordering. Always include `id`.
> - **Same error message not used for all cursor failures.** Don't leak whether a cursor was tampered vs. expired vs. from another user.

## References

- Stripe cursor pagination: https://stripe.com/docs/api/pagination
- "We need to talk about offset pagination": https://use-the-index-luke.com/no-offset
- Relay cursor connection spec (GraphQL): https://relay.dev/graphql/connections.htm
- PostgreSQL row value comparisons: https://www.postgresql.org/docs/current/functions-comparisons.html#ROW-WISE-COMPARISON

## Related

- [[Pagination Introduction]]
- [[Offset-limit Pagination]]
- [[Page-based Pagination]]
- [[Cursor Encryption]]
