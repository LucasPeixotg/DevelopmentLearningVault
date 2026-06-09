---
tags: [api-design, hateoas, hypermedia, rest, best-practices]
created: 2026-06-03
---
# Hypermedia and HATEOAS

> Summary: HATEOAS (Hypermedia As The Engine Of Application State) is a REST constraint where clients navigate an API through links discovered at runtime, not hardcoded URLs — the same way a browser follows links without knowing a site's URL structure in advance. Three competing formats exist: **HAL** (minimal, just `_links`), **JSON:API** (opinionated, covers data structure and relationships), and **Siren** (adds form-like actions). Almost no major public API implements full HATEOAS. Most land somewhere between "pagination links only" and "state-driven action links." The honest question is not whether to do HATEOAS but how much hypermedia is worth the cost.

## What it is

A HATEOAS response tells the client what it can do next. If an account is overdrawn, the server omits the `withdraw` link — the client doesn't need to know the business rule, it just checks whether the link is present. The business logic lives on the server, not duplicated in every client.

```json
{
  "id": "acc_42",
  "balance": -50,
  "_links": {
    "self":    { "href": "/accounts/acc_42" },
    "deposit": { "href": "/accounts/acc_42/transactions", "method": "POST" }
  }
}
```

No `withdraw` link because the account is overdrawn. The client never needs to know that rule.

## The three formats

| | HAL | JSON:API | Siren |
|---|---|---|---|
| **Complexity** | Low | High | Medium |
| **Describes write operations** | No | No | Yes |
| **Normalised resource graph** | No | Yes | No |
| **Pagination conventions** | No | Built-in | No |
| **Error format** | No | Built-in | No |
| **Adoption** | Moderate | Moderate | Very low |

**HAL** — adds `_links` and `_embedded` to JSON. Minimal; easy to retrofit onto an existing API.
```json
{
  "id": "ord_123", "amount": 2000,
  "_links": {
    "self":     { "href": "/v1/orders/ord_123" },
    "customer": { "href": "/v1/customers/cus_abc" },
    "refund":   { "href": "/v1/orders/ord_123/refunds" }
  }
}
```

**JSON:API** — opinionated structure: `data.type`, `data.attributes`, `data.relationships`, `included` for compound documents, sparse fieldsets via `?fields[orders]=amount`.
```json
{
  "data": {
    "type": "orders", "id": "ord_123",
    "attributes": { "amount": 2000 },
    "relationships": {
      "customer": { "data": { "type": "customers", "id": "cus_abc" } }
    }
  }
}
```

**Siren** — adds `actions` (form-like write descriptions). Useful for clients that need to know *how* to perform an operation, not just *that* it exists. Used rarely in production.

## When HATEOAS helps

> [!tip] Cases where it genuinely reduces coupling
> - **State-driven workflows** — valid operations change based on resource state (order `pending` vs `shipped`). Server encodes the state machine in the links it returns.
> - **Long-lived, independent teams** — server can add/move/remove operations without coordinating with the client team.
> - **Generic clients** — a client that must work against multiple APIs without prior knowledge (think: a web browser).
> - **Pagination links** — everyone does this. Even Stripe includes a `next` cursor. This is practical HATEOAS that almost all APIs already implement.

## When it adds complexity without value

> [!warning] Cases where the cost outweighs the benefit
> - **You control both client and server.** Decoupling benefit disappears; overhead remains.
> - **Typed clients generated from OpenAPI.** The client already has the URL structure baked in at generation time.
> - **Simple CRUD APIs.** The link graph is trivially predictable; generating and parsing links is pure overhead.
> - **Small teams who can coordinate.** What HATEOAS achieves through links, a 30-minute meeting achieves for free.

## The spectrum — where real APIs land

```
Level 0 — No hypermedia
Level 1 — Pagination links only                  ← Stripe, GitHub
Level 2 — Self-links on resources
Level 3 — Related resource links
Level 4 — State-driven action links              ← pragmatic sweet spot
Level 5 — Fully described actions (Siren-style)  ← Fielding's ideal, rarely seen
```

The pragmatic sweet spot for most APIs is **Level 3–4**: self-links on every resource, links to related resources, state-driven links where the business logic is complex enough to warrant it.

## Richardson Maturity Model

| Level | What it means |
|---|---|
| 0 | HTTP as a tunnel (SOAP, RPC style) |
| 1 | Resources — separate URLs per thing |
| 2 | HTTP verbs used correctly — `DELETE`, `404`, etc. |
| 3 | Hypermedia controls — response includes next-action links |

Most "REST APIs" are Level 2. Strict REST requires Level 3. Both labels are used loosely.

## Common mistakes

> [!warning] Anti-patterns
> - **Hardcoding URLs in the client while also adding `_links`.** You get the cost with none of the benefit.
> - **Including links for unavailable actions.** If a state transition isn't allowed, omit the link. Presence = available.
> - **Bare string link relation names for custom relations.** IANA standard relations (`self`, `next`, `collection`) are fine as strings. Custom relations must be URIs (`https://api.example.com/rels/refund`).
> - **Choosing JSON:API for its standardisation benefits but ignoring its verbosity cost.** Evaluate whether the overhead is justified for your use case.
> - **Treating HATEOAS as all-or-nothing.** Start with pagination links and self-links; add state-driven action links only where genuinely useful.

## References

- Roy Fielding's REST dissertation, Ch. 5: https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- HAL specification: https://datatracker.ietf.org/doc/html/draft-kelly-json-hal
- JSON:API specification: https://jsonapi.org/
- Siren specification: https://github.com/kevinswiber/siren
- Richardson Maturity Model (Martin Fowler): https://martinfowler.com/articles/richardsonMaturityModel.html
- IANA link relations registry: https://www.iana.org/assignments/link-relations/link-relations.xhtml

## Related

- [[API versioning strategies]]
- [[Pagination Introduction]]
- [[API Documentation]]
- [[OpenAPI Specification]]
- [[Error response formats]]
