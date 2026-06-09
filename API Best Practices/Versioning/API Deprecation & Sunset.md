---
tags:
  - deprecation
  - rest
  - best-practices
  - api-design
created: 2026-06-03
---

# API Deprecation & Sunset

> Summary: **Deprecation** = "this is going away." **Sunset** = "this dies on X date." Signal both machine-readably via `Deprecation`, `Sunset`, and `Link: rel="successor-version"` headers — not just in docs. Key takeaway: set a concrete sunset date and always include a `successor-version` link so clients can migrate programmatically.

## Endpoint lifecycle

1. Active → 2. Deprecated → 3. Sunset announced → 4. Sunset enforced (`410 Gone`)

## The standard headers

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: <https://api.example.com/docs/migration/v1-to-v2>; rel="sunset",
      <https://api.example.com/v2/charges/ch_abc123>; rel="successor-version"
```

- **`Deprecation`** (IETF draft, not yet final) — signals deprecation, optionally with a date
- **`Sunset`** (RFC 8594, finalized) — when the resource dies
- **`Link` with `rel="sunset"`** — points to migration docs
- **`Link` with `rel="successor-version"`** (RFC 5829) — points to the replacement
- **`Warning: 299`** — legacy, formally deprecated by RFC 9111 — don't use in new designs

> [!tip] Always include both Sunset and successor-version
> Clients need the date to schedule user-facing alerts; they need the URL to route traffic to the replacement. One without the other is incomplete.

> [!warning] Don't use Warning: 299
> Formally deprecated by RFC 9111. Use `Deprecation` + `Sunset` headers instead.

## After sunset

Return `410 Gone` (not `404`), ideally with a Problem Details body explaining where to go.

## Industry deprecation timelines

- Stripe: 3+ years, pinned versions effectively forever
- GitHub: ~24 months
- Google: minimum 12 months
- Startups: 3–6 months common

## Key anti-patterns

- Deprecating only in docs (no response headers)
- No sunset date
- Silent removal
- Sunset dates that keep slipping
- No migration path / no `successor-version`
- Not tracking who's still on the old version

## Related

- [[API Versioning Strategies]] — how to structure versions before you need to deprecate one

## References

- https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header — IETF Deprecation header draft
- https://www.rfc-editor.org/rfc/rfc8594 — RFC 8594: Sunset header
- https://www.rfc-editor.org/rfc/rfc5829 — RFC 5829: `successor-version` link relation
- https://www.rfc-editor.org/rfc/rfc9111 — RFC 9111: HTTP Caching (deprecated `Warning` header)
