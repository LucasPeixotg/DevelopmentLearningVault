---
name: rest-api-error-design
description: Use this skill when designing or implementing error responses for an HTTP/REST or GraphQL API — choosing an error format, building a global error handler, returning validation errors, or picking the right status code. Produces a consistent, machine-readable, security-safe error contract.
---

# REST API error design

## When to use this

You're adding error handling to an API: writing a global/exception handler, deciding
the shape of error bodies, returning validation failures, or reviewing what an API
leaks on failure. Errors are the most security-sensitive surface of an API and the
place clients integrate against most — get the contract right once and use it everywhere.

## Decision (pick a format)

- **New REST API → RFC 9457 Problem Details** (`application/problem+json`). This is the default.
- **Existing REST API with production clients →** keep your current envelope; changing
  the error format is a breaking change. Only migrate if you're versioning anyway.
- **GraphQL API →** the spec decides for you: always `200 OK`, errors in the top-level
  `errors` array, classification in `extensions.code`.

Whatever the format, every error must answer four questions: **what** happened
(machine-readable `type`/`code`), **why** (human `detail`/`message`), **where**
(field-level detail on validation), and **what to do next** (retry? fix input? quote a
`request_id`?).

## Implementation

### RFC 9457 Problem Details (the default)

```typescript
// problemDetails.ts
interface ProblemDetails {
  type: string;        // URI for the *class* of error — stable, clients switch on it
  title: string;       // short summary; MUST NOT vary between occurrences of one type
  status: number;      // repeated in body so it survives header stripping in logs
  detail?: string;     // explanation for *this* occurrence; may vary per request
  instance?: string;   // URI for this occurrence — quote in support tickets
  [key: string]: unknown; // custom extensions (request_id, errors[], code, ...) are allowed
}

export function sendProblem(res: Response, problem: ProblemDetails): void {
  res.status(problem.status)
     .contentType('application/problem+json')
     .json(problem);
}
```

The critical distinction: `type` + `title` describe the **class** of error (stable —
clients key on them); `detail` describes **this occurrence** (varies). If `title`
changes per request, clients can't rely on it.

### Global error handler — log full detail, return safe summary

```typescript
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error({ requestId: req.id, message: err.message, stack: err.stack,
                 userId: (req as any).user?.id, path: req.path, method: req.method });

  sendProblem(res, {
    type:       'https://api.example.com/errors/internal-error',
    title:      'An internal error occurred.',
    status:     500,
    request_id: req.id,   // the ONLY link between what the client sees and your logs
  });
});
```

Attach a correlation id early (`req.id = req.headers['x-request-id'] ?? randomUUID()`),
and include it in every error body, every log line, and outbound internal calls.

### Validation errors — collect all, return field-level detail

```typescript
if (errors.length > 0) {
  return sendProblem(res, {
    type:   'https://api.example.com/errors/validation-failed',
    title:  'Validation failed',
    status: 422,
    detail: 'One or more fields failed validation.',
    instance: req.path,
    errors,   // [{ field, code, message }] — never make clients fix one error at a time
  });
}
```

### GraphQL variant

Return `200 OK`; put errors in `errors[]` with classification in `extensions`:

```json
{ "data": { "user": {...}, "orders": null },
  "errors": [{ "message": "Insufficient permissions", "path": ["orders"],
    "extensions": { "code": "FORBIDDEN", "type": "https://api.example.com/errors/forbidden", "status": 403 } }] }
```

Use a fixed `code` enum (`NOT_FOUND`, `FORBIDDEN`, `UNAUTHENTICATED`, `VALIDATION_ERROR`,
`INTERNAL_ERROR`). Make optional relationships **nullable** so one resolver failure stays
local instead of nulling the whole response.

## Status code quick guide

| Status | Use for |
|---|---|
| `400` | malformed syntax / bad JSON / wrong content-type |
| `401` | missing or invalid credentials (unauthenticated) |
| `403` | authenticated but not authorised |
| `404` | not found — also use for private resources you're hiding (don't reveal with `403`) |
| `409` | state conflict (duplicate, optimistic-lock failure, idempotency in-progress) |
| `410` | intentionally removed (sunset endpoint) |
| `422` | well-formed but fails business validation |
| `429` | rate limited — always add `Retry-After` |
| `500` | unexpected — never leak detail |
| `503` | temporarily unavailable — add `Retry-After` |

`400` = the request is malformed; `422` = it's valid syntax but semantically wrong.

## Checklist

- [ ] One format used across every endpoint (clients can write generic handling).
- [ ] `Content-Type: application/problem+json` set on RFC 9457 responses.
- [ ] `type` is a URI; `title` is stable per type; occurrence detail is in `detail`.
- [ ] `request_id` / correlation id in every error body **and** every log line.
- [ ] Validation responses return **all** field errors at once with field-level detail.
- [ ] `401` for unauthenticated vs `403` for unauthorised; `404` (not `403`) for hidden resources.
- [ ] Auth failures return a generic "Invalid credentials" (no user-enumeration hints).

## Pitfalls

- **Leaking internals:** stack traces, DB error messages, internal hostnames/IPs, raw
  ORM exceptions. Log them internally; never send them. This is the #1 security mistake.
- **`{ "error": true }`** or `{ "message": "..." }` with no machine-readable code.
- **Inconsistent shapes across endpoints** — defeats generic client handling.
- **`200 OK` for errors** outside GraphQL — breaks HTTP semantics, caching, monitoring.
- **Same `title` and `detail`**, or occurrence-specific text in `title`.
- **`type` as a bare string code** (`"VALIDATION_ERROR"`) instead of a URI — add a `code`
  extension for readability, keep `type` a URI.
- **Returning one validation error at a time**; **no `request_id`** on errors.
- **GraphQL:** returning `4xx/5xx`, missing `extensions.code`, non-nullable everything.

## Reference

Vault notes (deep dives):
- `API Best Practices/Error Guidelines/RFC 7807 Problem Details.md`
- `API Best Practices/Error Guidelines/Error response formats.md`
- `API Best Practices/Error Guidelines/API error design guidelines.md`
- `API Best Practices/Error Guidelines/GraphQL error format.md`

External: RFC 9457 (https://datatracker.ietf.org/doc/html/rfc9457) ·
OWASP API Security Top 10 (https://owasp.org/API-Security/) ·
GraphQL error spec (https://spec.graphql.org/October2021/#sec-Errors)

Related skills: `api-pagination` (shares the error body on invalid cursor),
`api-rate-limiting` (429 + Retry-After), `express-production-api` (wires the handler in last).
