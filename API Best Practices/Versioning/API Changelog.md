---
tags: [api-design, documentation, best-practices, lifecycle]
created: 2026-06-03
---

# API Changelog

> Summary: An API changelog documents every change — breaking and non-breaking — in a dated, scannable format. Good changelog entries answer four questions: what changed, is it breaking, why, and what should the developer do. A well-maintained changelog is one of the strongest trust signals an API can have. Stripe's changelog is the industry benchmark.

## Anatomy of a good entry

Every entry must answer:

1. **What changed?** — specific endpoint, field, or behaviour.
2. **Is it breaking?** — explicit label, not buried in prose.
3. **Why?** — optional but builds trust.
4. **What should I do?** — a single actionable migration step.

```markdown
## 2025-11-14

### Breaking changes

#### POST /v1/charges — `currency` field now required

Previously, omitting `currency` defaulted to `usd`. This default has been
removed. Requests without an explicit `currency` field now return `422`.

**Who is affected:** integrations that omit `currency` in charge requests.
**Migration:** Add `"currency": "usd"` explicitly to all charge requests.
**Sunset:** Deprecated 2025-08-01, removed in this release.

---

### Non-breaking changes

#### GET /v1/customers — new `tax_exempt` field in response

Responses now include `tax_exempt` (`"none"`, `"exempt"`, `"reverse"`).
Existing integrations are unaffected.
```

Key decisions:
- **Breaking changes first, always labelled.** Developers scan for impact — help them skip irrelevant entries.
- **"Who is affected"** — one line that tells developers if this applies to them.
- **Migration step inline** — not just a link to a separate doc.
- **Newest first, date-stamped.**

## Changelog formats in the wild

**Stripe** (`stripe.com/docs/upgrades`):
- Date-based version names (`2024-10-28.acacia`).
- ⚠ Breaking label on breaking changes.
- Full history back to 2011 — single scrollable page.
- The changelog IS the API versioning mechanism — upgrading means moving to the next entry.

**GitHub** (`docs.github.com/rest/overview/breaking-changes`):
- Breaking changes page is separate from the general product changelog.
- Each entry includes deprecation date, removal date, and migration path.
- Emails developers who have called deprecated endpoints recently.

**Twilio** (`twilio.com/docs/usage/api-changelog`):
- Tag-based filtering by product area (Messaging, Voice, etc.).
- Separate "End of Life" notices from general changelog.

## The changelog as a trust signal

A maintained changelog signals:
- **Deliberate shipping** — changes are documented, not dropped silently.
- **Respect for consumer time** — breaking changes announced before they're felt.
- **Operational maturity** — the team has a process, not just a codebase.

An API with no changelog, or one that says "we made improvements" as an entry, signals the opposite. When evaluating an API to build on, changelog quality is a proxy for how much the team cares about developer experience.

## Common mistakes

> [!warning] Anti-patterns
> - **"We made some improvements."** Useless. Every entry should be specific enough that a developer can determine if it affects them.
> - **Burying breaking changes among non-breaking ones.** Breaking changes first, always.
> - **Changelog only on the website.** Put it in the SDK `CHANGELOG.md`, the API reference, and the developer portal.
> - **No changelog at all until something breaks.** Retroactive changelogs are never complete and lose trust.
> - **SDK and API changelogs that don't reference each other.** A developer upgrading the SDK should be able to find the corresponding API changelog section.

## References

- Stripe API upgrades (model changelog): https://stripe.com/docs/upgrades
- GitHub breaking changes: https://docs.github.com/en/rest/overview/breaking-changes
- "Keep a Changelog" conventions: https://keepachangelog.com/
- `oasdiff` (breaking change detection from OpenAPI diffs): https://github.com/tufin/oasdiff

## Related

- [[API Migration guides]]
- [[API lifecycle communication]]
- [[SDK versioning]]
- [[API deprecation and sunset]]
- [[API Versioning Strategies]]
