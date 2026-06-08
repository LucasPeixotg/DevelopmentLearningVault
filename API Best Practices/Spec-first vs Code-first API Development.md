---
tags: [api-design, documentation, openapi, best-practices, spec-first, code-first]
created: 2026-06-03
---
# Spec-first vs Code-first API Development

> Summary: Two workflows for producing an [[OpenAPI Specification]]. **Spec-first** writes the contract by hand, then generates stubs, SDKs, and mocks from it. **Code-first** annotates the server code and lets a library emit the spec. Spec-first wins for public APIs and parallel teams; code-first wins for internal services and solo iteration.

## The two workflows

**Code-first** — write the server, generate the spec from annotations/type hints. Tools: FastAPI, springdoc-openapi, NestJS Swagger, Swashbuckle.

**Spec-first** — write the spec, generate mocks/stubs/SDKs from it, build the server to match. Tools: Spectral (lint), Prism (mock), Redoc (docs), Speakeasy/Fern (SDKs), Schemathesis (contract tests).

## Comparison

| Concern                            | Spec-first                        | Code-first           |
| ---------------------------------- | --------------------------------- | -------------------- |
| Source of truth                    | Spec file                         | Server code          |
| Parallel team work                 | Day one                           | Blocked on backend   |
| Drift risk                         | Real — needs [[Contract Testing]] | None by construction |
| Design quality                     | Deliberate                        | Incidental           |
| Upfront effort                     | Higher                            | Lower                |
| Impl details leaking into contract | Low                               | High                 |

## When to choose

> [!tip] Default by audience
> 
> - **Public or partner API** → spec-first. The contract is the product.
> - **Internal microservice, single team** → code-first is fine.
> - **Frontend and backend are different teams** → spec-first pays for itself immediately.

> [!warning] Code-first trap Annotations cover types but rarely descriptions, examples, or error schemas. Treat the generated spec as a public artifact and polish it, or consumers will hate the docs.

Most mature API teams (Stripe, Twilio, GitHub) are spec-first. FastAPI's success shows code-first can work well when the framework does the heavy lifting. Hybrid is common: spec-first for design, code-first after — with contract tests guarding against drift.

## References

- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- Spectral linter: https://stoplight.io/open-source/spectral
- Arnaud Lauret, _The Design of Web APIs_ (Manning, 2019)

## Related

- [[OpenAPI Specification]]
- [[Contract Testing]]
- [[API versioning strategies]]
- [[AsyncAPI]]