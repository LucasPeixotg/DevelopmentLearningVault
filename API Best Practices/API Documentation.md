---
tags: [api-design, documentation, openapi, asyncapi, best-practices]
created: 2026-06-03
---
# API Documentation

> Summary: Good API docs are machine-readable specs that drive everything else (SDKs, mocks, tests, gateways) plus human-readable prose that explains intent. The two standards that matter: **OpenAPI** for request/response APIs (REST), **AsyncAPI** for event-driven APIs (Kafka, WebSockets, MQTT). The spec is the contract; the rendered docs are a derived artifact.

## The two standards

- **[[OpenAPI Specification]]** — describes REST APIs: endpoints, parameters, request/response shapes, auth, errors. The dominant standard. Most tooling (gateways, code generators, validators) assumes OpenAPI.
- **[[AsyncAPI]]** — same shape, different vocabulary. Describes channels and messages instead of paths and responses. Supports Kafka, AMQP, MQTT, WebSockets, NATS, Server-Sent Events, and others.

Both are YAML/JSON files. Both produce interactive HTML docs through renderers. Both can drive mocks, SDKs, and [[Contract Testing]]. Knowing one means you can read the other.

## What good docs include

- **Every endpoint/channel has a description** explaining _intent_, not just shape.
- **Realistic examples** for every request and response. Examples are the single highest-value thing for consumers.
- **All error responses documented**, not just the happy path — including status codes, error format, and what triggers each.
- **Authentication clearly specified** with `securitySchemes` (see [[Authentication and Authorization patterns]]).
- **Deprecation flags** on retiring endpoints, tying into [[API Deprecation & Sunset]].
- **Changelog** alongside the spec, so consumers can see what changed between versions.

## Rendering tools

Pick one — the spec is portable, the renderer is taste.

- **Swagger UI** — classic, interactive (try-it-out), busy.
- **Redoc** — clean three-column, read-only, popular for public docs.
- **Scalar** — modern, fast, good defaults.
- **Stoplight Elements** — embeddable, good for docs sites.

## Common mistakes

> [!warning] Anti-patterns
> 
> - **Docs as the only artifact.** If you're not generating mocks, SDKs, or validators from the spec, you're leaving most of its value unused.
> - **No examples.** Schema definitions alone don't teach consumers how to call the API.
> - **Documenting only the happy path.** Errors are where integrations actually break.
> - **Drift between spec and implementation.** Without [[Contract Testing]], the spec becomes a lie over time.
> - **Hand-written prose docs that duplicate the spec.** They drift, contradict, and rot. Generate from the spec where possible.

> [!tip] Where do the docs come from?
> See [[Spec-first vs Code-first API Development]] — the workflow you choose (spec-first vs code-first) determines who writes the spec, when, and how much drift you have to fight.

## References

- OpenAPI Specification: [https://spec.openapis.org/oas/latest.html](https://spec.openapis.org/oas/latest.html)
- AsyncAPI Specification: [https://www.asyncapi.com/docs/reference/specification/latest](https://www.asyncapi.com/docs/reference/specification/latest)
- Redoc: [https://redocly.com/redoc](https://redocly.com/redoc)
- Scalar: [https://scalar.com/](https://scalar.com/)

## Related

- [[OpenAPI Specification]]
- [[AsyncAPI]]
- [[Spec-first vs Code-first API Development]]
- [[API Deprecation & Sunset]]
- [[Authentication and Authorization patterns]]
- [[Contract Testing]]