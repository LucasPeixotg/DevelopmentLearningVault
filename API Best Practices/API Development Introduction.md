---
tags: [index, api-design, meta]
created: 2026-06-03
---

# API Development Introduction

> Summary: A decision guide through the notes in this vault, ordered by when each question typically arises. Each section presents the decision at stake. The notes do the arguing — this file does the routing.

---

## 1. Before writing any code

These decisions are expensive to reverse after clients integrate.

**How will clients signal which contract version they expect?**
Affects every URL you'll ever publish and how you route traffic.
- Comparing URI, header, query param, and date-based strategies: [[API versioning strategies]]

**What format will error responses take across every endpoint?**
Inconsistent errors force clients to write bespoke parsing per endpoint.
- Comparing RFC 7807, custom envelopes, and GraphQL errors: [[Error response formats]]
- Implementing the RFC 7807 standard: [[RFC 7807 Problem Details]]
- What to always include and never leak: [[API error design guidelines]]

**How will large collections be returned?**
Affects your database schema and query patterns, not just the API surface.
- Not sure which approach to use: [[Pagination Introduction]]
- Simple admin UI or small dataset: [[Offset-limit Pagination]]
- User-facing page numbers: [[Page-based Pagination]]
- Large or frequently mutating dataset: [[Cursor-based Pagination]]

**How will clients prove who they are?**
Each pattern suits different consumers and threat models.
- Overview of all patterns side by side: [[Authentication strategies compared]]
- Building a developer-facing API where the consumer is an account: [[API Keys]]
- Users granting third-party apps access to their data: [[OAuth 2.0]]
- Exchanging cryptographically signed tokens between services: [[JSON Web Tokens]]
- Mutual authentication between services you control: [[Mutual TLS]]

---

## 2. Before the first endpoint ships

**Who owns the API contract — the spec or the code?**
Determines how documentation, mocks, and SDKs are generated.
- Comparing the two workflows: [[Spec-first vs Code-first API Development]]
- Overview of documentation tools and formats: [[API Documentation]]
- Setting up OpenAPI: [[OpenAPI Specification]]
- Linting the spec in CI: [[Spectral]]
- Testing that the implementation matches the spec: [[Contract testing]]

**How will the API be protected from abusive or runaway clients?**
Limits added after launch may break clients who relied on having none.
- Understanding algorithms and headers: [[Rate limiting & throttling]]
- Comparing all Node.js approaches: [[Rate Limiting in Node and Typescript Comparison]]
- Single server or local dev: [[In-process Rate Limiting Implementation]]
- Multi-server, standard use case: [[Redis Libraries Rate Limiting Implementation]]
- Multi-server, need full algorithm control: [[Redis-backed sliding window rate limiting Implementation]]
- Already using a gateway: [[API Gateway Rate Limiting Implementation]]

---

## 3. When adding write operations

**Can clients safely retry a failed request without duplicating side effects?**
Every POST with a side effect (charge, email, provision) faces this question.
- Understanding idempotency and which HTTP methods are naturally safe: [[Idempotency]]
- The client-side key pattern and retry logic: [[Idempotency-Key header pattern]]
- Server-side storage, locking, and failure recovery: [[Idempotency server-side implementation]]

---

## 4. Before going to production

**Are the known vulnerabilities in your auth pattern addressed?**
Each auth method has a specific failure list worth reviewing before launch.
- JWT footguns and CVEs: [[JSON Web Tokens]]
- OAuth flow vulnerabilities: [[OAuth 2.0]]
- API key security: [[API Keys]]
- mTLS cert management: [[Mutual TLS]]

**Is the data inside your pagination cursors safe to expose?**
Base64 is obfuscation, not protection.
- Comparing levels from HMAC signing to AES-256-GCM: [[Cursor Encryption]]

**Do your services need to authenticate to each other at the transport layer?**
Depends on your threat model and regulatory requirements.
- Implementing mTLS between services: [[Mutual TLS]]
- Automating mTLS across many services: [[Service mesh]]

---

## 5. As the API evolves

**How will consumers learn about changes before something breaks?**
- Structuring and maintaining a changelog: [[API Changelog]]
- Communicating sunset dates via response headers: [[API Deprecation & Sunset]]

**How will consumers migrate when something does break?**
- Writing a guide that gets developers to act: [[API Migration guides]]
- Identifying and reaching holdouts before the cutoff: [[API lifecycle communication]]

**How do SDK releases stay aligned with API version changes?**
- SemVer, `@deprecated` annotations, version pinning: [[SDK versioning]]

---

## 6. Context-specific

**Documenting an event-driven or messaging API?**
REST specs don't cover Kafka, WebSockets, or MQTT: [[AsyncAPI]]

**Building a GraphQL API?**
Errors in GraphQL always return `200 OK` — different rules apply: [[GraphQL error format]]

**API operations depend on resource state (e.g. order `pending` vs `shipped`)?**
HATEOAS encodes the state machine in the links the server returns: [[Hypermedia and HATEOAS]]

**Multiple services calling each other in Kubernetes?**
A mesh handles mTLS, cert rotation, and traffic policy automatically: [[Service mesh]]

**Creating an API using express + Typescript?** 
Express middleware baselines: [[Express production middleware baseline]]

---

## All notes

| Topic         | Notes                                                                                                                                                                                                                                                                                               |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Versioning    | [[API versioning strategies]]                                                                                                                                                                                                                                                                       |
| Deprecation   | [[API Deprecation & Sunset]]                                                                                                                                                                                                                                                                        |
| Documentation | [[API Documentation]] · [[Spec-first vs Code-first API Development]] · [[OpenAPI Specification]] · [[AsyncAPI]] · [[Spectral]] · [[Contract testing]]                                                                                                                                               |
| Auth          | [[Authentication strategies compared]] · [[API Keys]] · [[OAuth 2.0]] · [[JSON Web Tokens]] · [[Mutual TLS]] · [[OpenID Connect]] · [[Service mesh]]                                                                                                                                                |
| Rate limiting | [[Rate limiting & throttling]] · [[Rate Limiting in Node and Typescript Comparison]] · [[In-process Rate Limiting Implementation]] · [[Redis-backed sliding window rate limiting Implementation]] · [[Redis Libraries Rate Limiting Implementation]] · [[API Gateway Rate Limiting Implementation]] |
| Errors        | [[Error response formats]] · [[RFC 7807 Problem Details]] · [[GraphQL error format]] · [[API error design guidelines]]                                                                                                                                                                              |
| Pagination    | [[Pagination Introduction]] · [[Offset-limit Pagination]] · [[Page-based Pagination]] · [[Cursor-based Pagination]] · [[Cursor Encryption]]                                                                                                                                                         |
| Idempotency   | [[Idempotency]] · [[Idempotency-Key header pattern]] · [[Idempotency server-side implementation]]                                                                                                                                                                                                   |
| Lifecycle     | [[API Changelog]] · [[API Migration guides]] · [[API lifecycle communication]] · [[SDK versioning]]                                                                                                                                                                                                 |
| Hypermedia    | [[Hypermedia and HATEOAS]]                                                                                                                                                                                                                                                                          |
