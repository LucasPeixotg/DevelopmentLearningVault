---
tags: [api-design, documentation, best-practices, lifecycle]
created: 2026-06-03
---
# API Migration guides

> Summary: A migration guide is not a list of what changed — that's the [[API Changelog]]. It is step-by-step instructions for moving from version A to version B, written for a developer who has been on v1 for two years and needs to upgrade in a weekend. Two things separate guides that get developers to migrate from those that don't: **before/after code examples for every breaking change**, and **a time estimate and deadline on the first screen**.

## Structure

```markdown
# Migrating from API v1 to v2

## Overview

**Time estimate:** 2–4 hours for a typical integration.
**Deadline:** v1 will be sunset on March 31, 2026.

[One paragraph on why v2 exists and what's better about it.]

## Breaking changes

### 1. [Change name]

**Before (v1):**
[code example]

**After (v2):**
[code example]

**Action:** [Single sentence — exactly what to do.]

---

### 2. [Next change]
...

## Testing your migration

[How to test in sandbox. What success looks like.]

## Timeline

| Date | Event |
|---|---|
| 2024-09-01 | v2 released |
| 2025-03-01 | v1 deprecated (headers sent on every response) |
| 2026-03-31 | v1 sunset — returns `410 Gone` |

## Getting help

[Support channel, dedicated migration email, office hours.]
```

## The two things that actually get developers to migrate

### 1. Before/after code examples for every breaking change

Prose descriptions of what changed are not enough. Developers need to see what to find in their code and what to replace it with:

```markdown
### Authentication: API keys now use Bearer scheme

Before (v1):
```http
Authorization: Token sk_live_abc123
```

After (v2):
```http
Authorization: Bearer sk_live_abc123
```

**Action:** Replace `Token` with `Bearer` in all Authorization headers.
```

Without the code example, a developer reads "authentication header format changed" and still has to figure out what that means for their codebase. With it, they do a find-and-replace and move on.

### 2. Time estimate and deadline on the first screen

Developers triage work. A migration guide without a time estimate and deadline in the first paragraph gets deprioritised indefinitely:

```markdown
## Overview

**Time estimate:** 2–4 hours for a typical integration.
**Deadline:** v1 will be sunset on March 31, 2026.
```

This is the first thing a developer needs to answer: "how big is this, and when does it need to be done?" Put it at the top, not buried in a timeline section at the bottom.

## What a migration guide is not

> [!warning] Common failures
> - **A changelog entry.** "Here's what changed" ≠ "here's how to migrate." Different documents, different purposes.
> - **A complete API reference for v2.** The guide only covers what changed. Link to the full reference; don't reproduce it.
> - **A document without deadlines.** An undated migration guide is treated as optional.
> - **Prose only.** Every breaking change needs a code example. Every. Single. One.
> - **One guide for all integrations.** If you have SDK users, REST users, and webhook consumers, they each have different migration paths. Separate guides (or clearly separated sections) per integration type.

## Scope considerations

> [!tip] One guide per breaking change cluster
> If v2 has 3 breaking changes and v3 has 5 different ones, maintain separate v1→v2 and v2→v3 guides. Don't force developers who are already on v2 to read the v1 migration content. A developer upgrading from v1 directly to v3 should be able to follow "v1→v2" then "v2→v3" in sequence.

## References

- Stripe migration guides (good examples): https://stripe.com/docs/upgrades
- Twilio migration guides: https://www.twilio.com/docs/usage/api

## Related

- [[API Changelog]]
- [[API lifecycle communication]]
- [[SDK versioning]]
- [[API Deprecation & Sunset]]
