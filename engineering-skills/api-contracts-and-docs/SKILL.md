---
name: api-contracts-and-docs
description: Use this skill when setting up API documentation or the contract workflow — deciding spec-first vs code-first, writing/maintaining an OpenAPI (or AsyncAPI) spec, linting it in CI, or guarding against spec/implementation drift with contract tests. Produces a spec that drives docs, SDKs, mocks, and tests.
---

# API contracts & docs

## When to use this

You're standing up documentation for an API, choosing how the spec and code relate, or
wiring the spec into CI (linting, mocks, contract tests). The spec is the **contract**; the
rendered docs are a *derived artifact*. A good machine-readable spec drives SDKs, mocks,
validators, and tests — treat docs as the least of what it produces.

## Decision

### Spec-first vs code-first

| | Spec-first | Code-first |
|---|---|---|
| Source of truth | the spec file | the server code |
| Parallel team work | day one | blocked on backend |
| Drift risk | real — needs contract tests | none by construction |
| Design quality | deliberate | incidental |
| Upfront effort | higher | lower |

Default by **audience**:
- **Public or partner API → spec-first.** The contract *is* the product.
- **Frontend and backend are different teams → spec-first** — it pays for itself immediately.
- **Internal microservice, single team → code-first** is fine (FastAPI/NestJS/springdoc
  generate the spec for you).
- **Hybrid is common:** spec-first for design, code-first after, with contract tests guarding drift.

### Which spec standard

- **OpenAPI** for request/response (REST) APIs — the dominant standard; most tooling assumes it.
- **AsyncAPI** for event-driven APIs (Kafka, AMQP, MQTT, WebSockets, SSE) — same shape,
  channels/messages instead of paths/responses.

## Implementation — the spec-first toolchain

1. **Write/own the OpenAPI spec** (`openapi.yaml`) as the source of truth.
2. **Lint in CI** with **Spectral** so style/quality rules are enforced on every PR:

   ```bash
   spectral lint openapi.yaml --fail-severity=warn
   ```

3. **Mock** from the spec with **Prism** so consumers build before the backend exists:

   ```bash
   prism mock openapi.yaml
   ```

4. **Generate SDKs** (Speakeasy / Fern) and **render docs** (Redoc, Scalar, Swagger UI,
   Stoplight Elements) from the same spec — pick a renderer by taste; the spec is portable.
5. **Contract-test** the implementation against the spec (e.g. Schemathesis) in CI so the
   running server can't drift from the contract. This is mandatory for spec-first, since
   drift is the one real risk of the approach.

### What "good docs" must include

- Every endpoint/channel has a description of **intent**, not just shape.
- **Realistic examples** for every request and response — the single highest-value thing
  for consumers.
- **All error responses** documented (status, format, trigger), not just the happy path.
- Authentication declared via `securitySchemes`.
- **Deprecation flags** on retiring endpoints (ties into `api-versioning-lifecycle`).
- A **changelog** alongside the spec.

## Checklist

- [ ] Workflow chosen by audience (spec-first for public/multi-team; code-first for internal).
- [ ] One machine-readable spec is the source of truth (OpenAPI for REST, AsyncAPI for events).
- [ ] Spec linted in CI (Spectral) and failing the build on violations.
- [ ] Contract tests run in CI if spec-first (or hybrid) — no silent drift.
- [ ] Examples on every request/response; all error responses documented.
- [ ] `securitySchemes` defined; deprecation flags present; changelog beside the spec.
- [ ] Spec actually drives mocks/SDKs/validators — not just a rendered docs page.

## Pitfalls

- **Docs as the only artifact** — leaving SDK/mock/validator value on the table.
- **No examples** — schemas alone don't teach consumers how to call the API.
- **Documenting only the happy path** — errors are where integrations break.
- **Drift between spec and implementation** — without contract tests the spec becomes a lie.
- **Hand-written prose that duplicates the spec** — it rots and contradicts; generate instead.
- **Code-first trap:** annotations cover types but rarely descriptions, examples, or error
  schemas — polish the generated spec or consumers hate the docs.

## Reference

Vault notes:
- `API Best Practices/Spec-first vs Code-first API Development.md`
- `API Best Practices/API Documentation.md`

External: OpenAPI Specification (https://spec.openapis.org/oas/latest.html) ·
AsyncAPI (https://www.asyncapi.com/docs/reference/specification/latest) ·
Spectral (https://stoplight.io/open-source/spectral) · Redoc (https://redocly.com/redoc) ·
Scalar (https://scalar.com/)

Related skills: `api-versioning-lifecycle` (deprecation flags + changelog in the spec),
`rest-api-error-design` (the error schemas the spec documents),
`api-authentication` (the `securitySchemes` you declare).
