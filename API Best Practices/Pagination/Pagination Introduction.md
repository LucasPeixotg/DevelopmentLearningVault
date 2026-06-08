---
tags: [api-design, pagination, best-practices]
created: 2026-06-03
---
# Pagination Introduction

> Summary: Pagination splits large datasets into manageable chunks. The choice of algorithm affects performance at scale, consistency under mutations, and whether clients can navigate arbitrarily or only sequentially. Three approaches: **offset/limit** (simple, breaks at depth), **page-based** (same as offset with different math), **cursor-based** (consistent, scalable, no random access). Most public APIs at scale use cursor-based.

## Why pagination matters

Returning all records in one response doesn't scale. `GET /orders` across 10 million rows will time out, exhaust server memory, and destroy the client parsing it. Pagination is also the first place APIs degrade silently under load — page 1 is fast, page 5000 is not (depending on algorithm).

The choice affects:

- **Performance** — does `OFFSET 10000000` cost more than `OFFSET 20`?
- **Consistency** — does inserting a row mid-pagination cause duplicates or skips?
- **Navigability** — can the client jump to page 47, or only go forward?
- **Ergonomics** — how much client complexity does it add?

## At a glance

|                                | [[Offset-limit Pagination\|Offset/limit]] | [[Page-based Pagination\|Page-based]] | [[Cursor-based Pagination\|Cursor-based]] |
| ------------------------------ | ----------------------------------------- | ------------------------------------- | ----------------------------------------- |
| **Random access**              | Yes                                       | Yes                                   | No                                        |
| **Total count**                | Cheap                                     | Cheap                                 | Expensive                                 |
| **Performance at depth**       | Degrades                                  | Degrades                              | Constant                                  |
| **Consistent under mutations** | No                                        | No                                    | Yes                                       |
| **Implementation complexity**  | Low                                       | Low                                   | Medium                                    |
| **Best for**                   | Admin UIs, small data                     | Same as offset/limit                  | Large data, feeds, public APIs            |

## Response envelope design

All three approaches need a consistent envelope shape. Pick one and use it everywhere.

**Cursor-based envelope:**
```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "cursor_Y3JlYXRl",
    "hasMore": true
  }
}
```

**Offset-based envelope:**
```json
{
  "data": [...],
  "pagination": {
    "offset": 40,
    "limit":  20,
    "total":  1543
  }
}
```

## Should you include `totalCount`?

- **Offset/limit** — `totalCount` is cheap (one `COUNT(*)`) and often expected by UIs showing "1,543 results."
- **Cursor-based** — `totalCount` requires a full table scan on every page. Most cursor-based APIs omit it (Stripe, GitHub). If you genuinely need it, expose a separate `GET /orders/count` endpoint or return it only on the first page.

Committing to `totalCount` on every cursor-paginated response is a trap — you pay the cost on every request, or return stale counts.

## The Link header approach

Some APIs (GitHub, older REST APIs) use `Link` headers instead of a body envelope:

```http
Link: <https://api.example.com/orders?after=cursor_abc>; rel="next",
      <https://api.example.com/orders?after=cursor_xyz>; rel="prev"
```

The `rel` values (`next`, `prev`, `first`, `last`) are standardised in RFC 5988. It's more RESTful but harder to parse, can't carry `totalCount` or `hasMore`, and even GitHub is moving away from it. Body envelopes are universally preferred in modern APIs.

## Common mistakes (across all approaches)

> [!warning] Anti-patterns
> - **No default limit.** `GET /orders` with no limit returns all records. Always have a default (e.g. 20) and a maximum (e.g. 100).
> - **Accepting unbounded `limit`.** Validate and cap it. `?limit=1000000` should return `400`.
> - **Inconsistent sort order.** Without a stable, deterministic ORDER BY (including a unique tiebreaker), cursors are unreliable and offset results are non-deterministic.
> - **Inconsistent envelope shape across endpoints.** Clients can't write generic pagination handling if `/orders` and `/customers` use different shapes.
> - **No handling for the empty last page.** When there's no more data, `hasMore: false` and `nextCursor: null` should be explicit, not absent.

## References

- Stripe pagination docs (cursor-based): https://stripe.com/docs/api/pagination
- GitHub pagination docs (Link header): https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api
- "We need to talk about offset pagination": https://use-the-index-luke.com/no-offset
- RFC 5988 (Link relations): https://datatracker.ietf.org/doc/html/rfc5988

## Related

- [[Offset-limit Pagination]]
- [[Page-based Pagination]]
- [[Cursor-based Pagination]]
- [[Cursor Encryption]]
- [[API Documentation]]
- [[Error response formats]]
