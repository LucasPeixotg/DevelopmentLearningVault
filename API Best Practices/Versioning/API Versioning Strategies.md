---
tags:
  - versioning
  - rest
  - best-practices
  - api-design
created: 2026-06-03
---

# API Versioning Strategies

> Summary: Four strategies for versioning HTTP APIs — URI path, custom header, media type, and query param — each with different tradeoffs around visibility, cacheability, and REST purity. Date-based versioning (Stripe-style) is orthogonal to these four and is the approach that best supports continuous evolution with per-customer pinning.

## Breaking vs. non-breaking changes

- Breaking: removing/renaming fields, changing types, making optional fields required
- Non-breaking: adding optional fields, adding new endpoints
- Postel's Law: be conservative in what you send, liberal in what you accept

## The four strategies

| Strategy          | Example                                   | Pros                                           | Cons                                                                  | Used by                   |
| ----------------- | ----------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------- | ------------------------- |
| **URI**           | `GET /v2/charges`                         | Visible, easy to route, cache-friendly, simple | Violates REST purism, breaks hyperlinks, encourages big-bang releases | GitHub, Twitter, Facebook |
| **Custom header** | `API-Version: 2024-10-01`                 | Stable URLs, supports per-customer pinning     | Invisible in logs, harder to debug, needs `Vary` for caching          | Stripe, Twilio            |
| **Media type**    | `Accept: application/vnd.example.v2+json` | Most RESTful, stable URLs                      | Awkward syntax, uneven tooling                                        | GitHub (older)            |
| **Query param**   | `?version=2`                              | Easy to test in browser                        | Mixes versioning with filters, considered least clean                 | Older AWS, internal APIs  |

## Date-based versioning

(e.g., Stripe's `2024-10-28.acacia`): orthogonal to the four above. Versions are dates, naturally ordered. Enables continuous evolution and per-customer pinning, but requires expensive server-side compatibility shims.

## Common Mistakes

> [!warning] Anti-patterns
> 
> - Versioning every endpoint independently
> - Releasing v2 for cosmetic fixes
> - No deprecation policy
> -  Mixing strategies (URL + header that disagree)

## Related

- [[API Deprecation & Sunset]] — read this next: how to communicate and enforce the end-of-life of a version once you've committed to retiring it
- [[API Changelog]]
