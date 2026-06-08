---
tags: [api-design, error-handling, security, best-practices]
created: 2026-06-03
---
# API error design guidelines

> Summary: Format-agnostic guidelines for designing error responses — applies equally to [[RFC 7807 Problem Details]], custom envelopes, and [[GraphQL error format]]. Core principle: **log the full detail internally, return the safe summary externally**, linked by a correlation ID. Errors are the most security-sensitive part of your API surface — stack traces and DB messages are a roadmap for attackers.

## What to always include

| Field | Purpose | Example |
|---|---|---|
| Machine-readable code/type | Clients switch on this, not the message | `"type": "https://api.example.com/errors/validation-failed"` |
| Human-readable message | For developers reading logs or building integrations | `"detail": "Amount must be positive"` |
| HTTP status in body | Visible in logs when headers are stripped | `"status": 422` |
| Correlation / request ID | Client quotes this to support; you search your logs for it | `"request_id": "req_abc123"` |
| Field-level detail on validation errors | Without this, clients show users "something went wrong" | `"errors": [{ "field": "amount", "message": "..." }]` |

## What to never include

> [!warning] Never send these to clients
> - **Stack traces.** Reveal file paths, library versions, internal architecture — a roadmap for attackers.
> - **Database error messages.** `"duplicate key value violates unique constraint 'users_email_key'"` exposes your schema and DB engine.
> - **Internal service names or IPs.** `"Could not connect to billing-service.internal:5432"`.
> - **Raw ORM/exception messages.** Often contain file paths, class names, SQL queries.
> - **Enumeration hints on auth endpoints.** `"User alice@example.com not found"` confirms the email exists. Return `"Invalid credentials"` for both wrong-email and wrong-password cases.

## The logging pattern

Log everything internally, return only the safe summary:

```typescript
// Global error handler (Express)
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {

  // Full detail — internal only, never sent to client
  logger.error({
    requestId: req.id,
    message:   err.message,
    stack:     err.stack,
    userId:    (req as any).user?.id,
    path:      req.path,
    method:    req.method,
  });

  // Safe summary — sent to client
  res.status(500)
    .contentType('application/problem+json')
    .json({
      type:       'https://api.example.com/errors/internal-error',
      title:      'An internal error occurred.',
      status:     500,
      request_id: req.id,  // client quotes this when filing a support ticket
    });
});
```

The `request_id` is the only link between what the client sees and what you log.

## Validation errors

Validation errors are the most common case. Always include field-level detail:

```typescript
// Collect all field errors before returning — don't return one at a time
function validateCharge(body: unknown): ValidationError[] {
  const errors: ValidationError[] = [];

  if (!body.amount || body.amount <= 0) {
    errors.push({ field: 'amount', code: 'INVALID_VALUE', message: 'Must be a positive integer.' });
  }
  if (!body.currency || !/^[A-Z]{3}$/.test(body.currency)) {
    errors.push({ field: 'currency', code: 'INVALID_FORMAT', message: 'Must be a valid ISO 4217 code.' });
  }

  return errors;
}

// Return all errors at once, not just the first one
if (errors.length > 0) {
  return sendProblem(res, {
    type:     'https://api.example.com/errors/validation-failed',
    title:    'Validation failed',
    status:   422,
    detail:   'One or more fields failed validation.',
    instance: req.path,
    errors,
  });
}
```

> [!tip] Return all validation errors at once
> Returning only the first error forces clients to fix-and-retry multiple times. Collect all errors, return them in one response.

## Correlation IDs

Every request should get a unique ID, attached early in the middleware chain:

```typescript
import { randomUUID } from 'crypto';

app.use((req, _res, next) => {
  // Re-use an upstream trace ID if present (API gateway, load balancer)
  req.id = req.headers['x-request-id'] as string ?? randomUUID();
  next();
});
```

Include it in:
- Every error response (`request_id` field)
- Every log line
- Every outgoing request to internal services (`X-Request-Id` header)
- Response headers (`X-Request-Id` or `Request-Id`) so clients can capture it

## HTTP status code guide for errors

| Status | When to use |
|---|---|
| `400 Bad Request` | Malformed request syntax, missing required fields |
| `401 Unauthorized` | Missing or invalid credentials (despite the name: unauthenticated) |
| `403 Forbidden` | Authenticated but not authorised for this action |
| `404 Not Found` | Resource doesn't exist — or exists but you're hiding it |
| `409 Conflict` | State conflict (duplicate resource, optimistic lock failed) |
| `410 Gone` | Resource intentionally removed (deprecated endpoint) |
| `422 Unprocessable Entity` | Request is well-formed but fails business validation |
| `429 Too Many Requests` | Rate limit exceeded — always include `Retry-After` |
| `500 Internal Server Error` | Unexpected server error — never leak detail |
| `503 Service Unavailable` | Server temporarily unavailable — include `Retry-After` |

> [!warning] `400` vs `422`
> `400` = the request itself is malformed (bad JSON, wrong Content-Type). `422` = the request is syntactically valid but semantically wrong (amount is negative, email is already taken). Many APIs use `400` for both — acceptable, but `422` is more precise for validation failures.

## Common mistakes

> [!warning] Anti-patterns
> - **`{ "error": true }`** — tells the client nothing.
> - **Inconsistent formats across endpoints.** Clients can't write generic error handling.
> - **`200 OK` for errors** (outside GraphQL). Breaks HTTP semantics, caching, monitoring.
> - **Stack traces in production.** The single most common security mistake.
> - **Same `title` and `detail`.** Title = stable class descriptor. Detail = occurrence-specific context.
> - **Leaking `404` vs `403` for private resources.** Return `404` for both to avoid confirming a resource exists.
> - **Returning one validation error at a time.** Forces clients into painful fix-and-retry cycles.
> - **No `request_id`.** Support tickets without a reference ID are unresolvable.

## References

- RFC 9457 (Problem Details): https://datatracker.ietf.org/doc/html/rfc9457
- OWASP API Security Top 10: https://owasp.org/API-Security/
- Zalando error response guidelines: https://opensource.zalando.com/restful-api-guidelines/#176

## Related

- [[Error response formats]]
- [[RFC 7807 Problem Details]]
- [[GraphQL error format]]
- [[Rate limiting & throttling]]
- [[Authentication Strategies Compared]]
