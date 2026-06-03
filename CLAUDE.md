# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Repository Overview

This is an Obsidian vault used as a personal learning workspace — not a software project. There is no build system, test suite, or application code. The primary goal is capturing and organizing development learnings: API best practices, patterns, gotchas, and reference material.

## Vault Structure

- `.obsidian/` — Obsidian app configuration (plugins, appearance, workspace). Don't edit by hand unless fixing a known config issue.
- `*.md` — Markdown notes using CommonMark + Obsidian wiki-link syntax.
- `*.base` — Obsidian Base files (structured table/database views). YAML front matter drives the view type.
- `Attachments/` — Default folder for embedded files (images, PDFs).

## Obsidian-Specific Syntax

|Syntax|Purpose|
|---|---|
|`[[Note Title]]`|Internal link|
|`[[Note Title\|display text]]`|Internal link with alias|
|`![[Note Title]]`|Embed another note|
|`![[image.png]]`|Embed an image|
|`#tag`|Inline tag|
|`tags: [tag1, tag2]`|YAML front matter tags|

No lint or format commands — content is freeform Markdown.

## Note Conventions

When creating or editing notes, follow these conventions:

**YAML front matter** (include on all notes):

```yaml
---
tags: [api, rest, best-practices]   # kebab-case, plural categories first
created: YYYY-MM-DD
source: https://...                  # original reference if applicable
---
```

**Summary first**: Every note opens with a short summary (1–3 sentences) directly under the title, before any other heading. It should state what the note covers and why it matters, so it's readable at a glance and useful in link previews and search. Example:

```markdown
# Idempotency in REST APIs

> Summary: How to make POST/PATCH requests safely retryable using idempotency keys, and why GET/PUT are idempotent by definition. Key takeaway: generate the key client-side and have the server dedupe.

## Why it matters
...
```

**Headings**: Use sentence case (`## Error handling`, not `## Error Handling`).

**Code blocks**: Always include the language identifier (` ```python `, ` ```bash `, etc.).

**Callouts** (Obsidian-native, use for emphasis):

```
> [!tip] When to use this
> Short, actionable context.

> [!warning] Watch out
> Common pitfall or gotcha.
```

## Linking to Existing Notes

This is a graph, not a folder of isolated files — links are what make it useful. When creating or editing a note:

- **Always check for existing related notes first.** Before writing, look through the vault for notes on adjacent topics (search by keyword, scan filenames and tags). Link to them rather than re-explaining a concept that already has its own note.
- **Add `[[wiki-links]]` inline** wherever you reference a concept that has (or should have) its own note. Prefer `[[Note Title]]` over bare URLs for internal concepts.
- **End each note with a `## Related` section** listing `[[links]]` to connected notes — prerequisites, deeper dives, contrasting approaches.
- **Surface missing connections.** If you notice an existing note that _should_ link to the one being edited (or vice versa), point it out so the backlink can be added.
- **Reuse existing tags** consistent with the current tag vocabulary rather than inventing near-duplicates (`#rest-api` vs `#rest`).

Use bare URLs only for external references — put those in a `## References` section at the bottom.

## Content Goals

This vault documents _development learnings_, so notes should be:

- **Opinionated** — capture what actually worked, not just what the docs say
- **Scannable** — summary, then bullets or code, then caveats
- **Linked** — connect related concepts so the graph is useful
- **Evergreen where possible** — prefer timeless patterns over version-specific details; note versions explicitly when relevant (e.g., "as of OpenAI API v2")

## What Claude Should and Shouldn't Do

**Do:**

- Help draft, improve, or restructure notes
- Search the vault for related notes and add `[[links]]` to them
- Suggest backlinks from existing notes when relevant
- Propose tags consistent with existing tag conventions
- Summarize external content into note format on request
- Refactor verbose notes into tighter, more scannable versions

**Don't:**

- Create files outside the vault root or existing folders without asking
- Rename files without confirming (Obsidian links break if the file moves without a redirect)
- Edit `.obsidian/` config files unless explicitly asked
- Add front matter fields not listed above without asking