---
name: api-versioning-lifecycle
description: Use this skill when versioning a published API, making a breaking change, deprecating or sunsetting an endpoint/version, or keeping SDKs and changelogs in sync. Produces a versioning strategy and a machine-readable deprecation path clients can act on.
---

# API versioning & lifecycle

## When to use this

You're about to publish an API and need a versioning strategy, you're making a change that
might break clients, or you're retiring an old version/endpoint. These decisions are
expensive to reverse once clients integrate — get the strategy and the deprecation signals
right before, not after.

## Decision

### Is the change even breaking?

- **Breaking:** removing/renaming fields, changing types, making an optional field required.
- **Non-breaking:** adding optional fields, adding new endpoints.
- Follow **Postel's Law:** conservative in what you send, liberal in what you accept.

Only breaking changes need a new version. Don't ship v2 for cosmetic fixes.

### Versioning strategy

| Strategy | Example | Trade-off | Used by |
|---|---|---|---|
| **URI path** | `GET /v2/charges` | visible, cache-friendly, simple; encourages big-bang releases | GitHub, Twitter |
| **Custom header** | `API-Version: 2024-10-01` | stable URLs, per-customer pinning; invisible in logs, needs `Vary` | Stripe, Twilio |
| **Media type** | `Accept: application/vnd.example.v2+json` | most RESTful; awkward, uneven tooling | GitHub (older) |
| **Query param** | `?version=2` | easy to test; mixes versioning with filters | older AWS |

- **Default to URI path versioning** for most APIs — visible, trivial to route and cache.
- **Date-based versioning** (Stripe's `2024-10-28.acacia`) is orthogonal and best for
  continuous evolution with per-customer pinning — but you pay for server-side
  compatibility shims forever. Reach for it only if you need that.
- Don't version every endpoint independently, and never mix strategies that can disagree.

## Implementation — the deprecation path

**Deprecation** = "this is going away." **Sunset** = "this dies on date X." Signal both
**machine-readably in response headers**, not just in docs. Lifecycle:
`Active → Deprecated → Sunset announced → Sunset enforced (410 Gone)`.

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: <https://api.example.com/docs/migration/v1-to-v2>; rel="sunset",
      <https://api.example.com/v2/charges/ch_abc123>; rel="successor-version"
```

- `Deprecation` (IETF draft) — signals deprecation, optionally dated.
- `Sunset` (RFC 8594) — when the resource dies.
- `Link rel="sunset"` → migration docs; `Link rel="successor-version"` (RFC 5829) → the replacement.
- **Always include both a concrete Sunset date and a `successor-version` link** — one
  without the other is incomplete (clients need the date to schedule alerts and the URL to
  reroute).
- After sunset, return **`410 Gone`** (not `404`), ideally with a Problem Details body
  pointing to the successor.
- **Do not** use `Warning: 299` — formally deprecated by RFC 9111.

Pick a deprecation window and hold it (Stripe 3+ yrs; GitHub ~24 mo; Google ≥12 mo;
startups 3–6 mo). **Track who's still on the old version** so you can reach holdouts before
the cutoff.

### Keep the ecosystem in sync

- **Changelog** alongside the spec — every breaking change, with dates and migration links.
- **Migration guides** that get developers to *act*: before/after payloads, the deadline,
  copy-paste diffs.
- **SDK versioning** with SemVer — bump major on breaking API changes, mark old methods
  `@deprecated`, support version pinning so an API change doesn't force an SDK upgrade.

## Checklist

- [ ] Versioning strategy chosen and applied consistently (no mixed/ disagreeing schemes).
- [ ] New version only for genuinely breaking changes; additions stay in place.
- [ ] Deprecated endpoints emit `Deprecation` + `Sunset` + `successor-version` headers.
- [ ] Concrete sunset date set and not silently slipped.
- [ ] `410 Gone` (with successor link) after sunset, not `404` or silent removal.
- [ ] Changelog + migration guide published; SDK majors bumped with `@deprecated` markers.
- [ ] You can identify which consumers still use the retiring version.

## Pitfalls

- **Deprecating only in docs** — clients can't detect it programmatically.
- **No sunset date**, or **dates that keep slipping** — clients never migrate.
- **Silent removal** / no `successor-version` — clients break with no path forward.
- **Versioning every endpoint independently**, or **v2 for cosmetic fixes**.
- **`Warning: 299`** for deprecation (use `Deprecation`/`Sunset`).
- **`404` instead of `410`** after sunset — looks like a transient bug, not an EOL.
- **SDK out of sync** — API breaks but the SDK still advertises the old shape.

## Reference

Vault notes:
- `API Best Practices/Versioning/API Versioning Strategies.md`
- `API Best Practices/Versioning/API Deprecation & Sunset.md`
- `API Best Practices/Versioning/API Changelog.md`
- `API Best Practices/Versioning/API Migration guides.md`
- `API Best Practices/Versioning/API lifecycle communication.md`
- `API Best Practices/Versioning/SDK versioning.md`

External: RFC 8594 Sunset (https://www.rfc-editor.org/rfc/rfc8594) ·
RFC 5829 successor-version (https://www.rfc-editor.org/rfc/rfc5829) ·
IETF Deprecation header draft (https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header)

Related skills: `rest-api-error-design` (the 410 Gone body), `api-contracts-and-docs`
(changelog lives with the spec).
