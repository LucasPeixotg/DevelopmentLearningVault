# Engineering skills

Reusable, action-oriented API engineering skills distilled from the notes in this
vault (`API Best Practices/`). Where the vault notes are the long-form **reference**
(what a concept is and why), these skills are the **playbook** (what to do, with the
opinionated default, copy-pasteable TypeScript/Express code, and the pitfalls to
avoid). Each skill links back to its source notes for the deep dive.

## Skills

| Skill | Use it when… | Source notes |
|---|---|---|
| [`rest-api-error-design`](rest-api-error-design/SKILL.md) | designing or adding error responses to a REST or GraphQL API | RFC 7807 Problem Details · Error response formats · API error design guidelines · GraphQL error format |
| [`api-pagination`](api-pagination/SKILL.md) | an endpoint returns a list / collection | Pagination Introduction · Offset-limit · Page-based · Cursor-based · Cursor Encryption |
| [`api-idempotency`](api-idempotency/SKILL.md) | adding a write endpoint with side effects (charge, email, provision) that clients may retry | Idempotency · Idempotency-Key header pattern · Idempotency server-side implementation |
| [`api-rate-limiting`](api-rate-limiting/SKILL.md) | protecting an API from abusive or runaway clients | Rate limiting & throttling · Node/TS comparison · In-process · Redis libraries · Redis sliding window · API gateway |
| [`api-authentication`](api-authentication/SKILL.md) | choosing or implementing how clients prove identity | Authentication strategies compared · API Keys · OAuth 2.0 · JSON Web Tokens · Mutual TLS · OpenID Connect · Service mesh |
| [`api-versioning-lifecycle`](api-versioning-lifecycle/SKILL.md) | versioning, deprecating, or evolving a published API | API versioning strategies · Deprecation & Sunset · Changelog · Migration guides · lifecycle communication · SDK versioning |
| [`api-contracts-and-docs`](api-contracts-and-docs/SKILL.md) | deciding spec-first vs code-first, or setting up API docs + contract tests | Spec-first vs Code-first · API Documentation · OpenAPI · AsyncAPI · Spectral · Contract testing |
| [`express-production-api`](express-production-api/SKILL.md) | scaffolding or hardening an Express + TypeScript API | Express production middleware baseline · CORS · Hypermedia and HATEOAS |

## Activating these as live Claude Code skills

These live in `engineering-skills/` so they travel with the vault. Claude Code only
auto-discovers skills under a `.claude/skills/` directory (project or user level), so
to make one trigger automatically, symlink or copy its directory there:

```bash
# User-level (available in every project):
ln -s "$(pwd)/engineering-skills/api-idempotency" ~/.claude/skills/api-idempotency

# Or all of them at once:
for d in engineering-skills/*/; do
  ln -s "$(pwd)/$d" ~/.claude/skills/"$(basename "$d")"
done
```

Then start a fresh session — the skill appears in the available-skills list and its
`description` (the trigger) decides when it fires.

## Conventions

- **Stack:** TypeScript + Express 5, matching the vault's implementation notes.
- **Format:** every skill is one `SKILL.md` with `name` + trigger-style `description`
  frontmatter, then *When to use → Decision → Implementation → Checklist → Pitfalls →
  Reference*.
- **Opinionated:** each leads with a default ("use RFC 9457", "default limit 20 / max
  100", "generate idempotency keys client-side"), the same stance as the vault.
