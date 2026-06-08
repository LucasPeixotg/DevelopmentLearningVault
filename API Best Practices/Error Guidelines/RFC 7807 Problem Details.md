---
tags: [api-design, error-handling, http, standards]
created: 2026-06-03
---
# RFC 7807 Problem Details

> Summary: The standard JSON format for HTTP error responses (RFC 7807, updated by RFC 9457 in 2023). Content type: `application/problem+json`. Defines five standard fields — `type`, `title`, `status`, `detail`, `instance` — and explicitly allows custom extensions. Use this for all new REST APIs. The key design insight: `type`+`title` describe the *class* of error (stable, client switches on these); `detail` describes *this occurrence* (varies per request).

## The standard fields

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type":     "https://api.example.com/errors/validation-failed",
  "title":    "Validation failed",
  "status":   422,
  "detail":   "The 'amount' field must be a positive integer.",
  "instance": "/v1/charges/req_abc123"
}
```

| Field | Required | Purpose |
|---|---|---|
| `type` | Recommended | URI identifying the *kind* of problem. Should be dereferenceable — pointing to docs is ideal. |
| `title` | Recommended | Short summary. **Must not change between occurrences** of the same `type`. |
| `status` | Recommended | HTTP status code repeated in the body — visible in logs even when headers are stripped. |
| `detail` | Optional | Explanation for *this specific occurrence*. May vary per request. |
| `instance` | Optional | URI identifying *this specific occurrence* — useful for support tickets. |

### `type` vs. `detail` — the critical distinction

- `type` + `title` = the **class** of error. Stable. Clients switch on `type` programmatically.
- `detail` = **this occurrence**. Varies. For human consumption and logs.

If your `title` changes between requests for the same error kind, clients can't rely on it.

## Custom extensions

RFC 7807 explicitly allows additional fields alongside the standard ones:

```json
{
  "type":     "https://api.example.com/errors/validation-failed",
  "title":    "Validation failed",
  "status":   422,
  "detail":   "One or more fields failed validation.",
  "instance": "/v1/charges/req_abc123",
  "request_id": "req_abc123",
  "errors": [
    {
      "field":   "amount",
      "code":    "INVALID_VALUE",
      "message": "Must be a positive integer."
    },
    {
      "field":   "currency",
      "code":    "INVALID_FORMAT",
      "message": "Must be a valid ISO 4217 code."
    }
  ]
}
```

`errors`, `request_id`, and any domain-specific fields are custom extensions — fully allowed and common in production APIs.

## TypeScript implementation (Express)

```typescript
// problemDetails.ts
interface ProblemDetails {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  [key: string]: unknown; // custom extensions
}

export function sendProblem(
  res: Response,
  problem: ProblemDetails
): void {
  res
    .status(problem.status)
    .contentType('application/problem+json')
    .json(problem);
}

// Usage in a route
app.post('/v1/charges', (req, res) => {
  if (!req.body.amount || req.body.amount <= 0) {
    return sendProblem(res, {
      type:       'https://api.example.com/errors/validation-failed',
      title:      'Validation failed',
      status:     422,
      detail:     "The 'amount' field must be a positive integer.",
      instance:   req.path,
      request_id: req.id,
      errors: [
        { field: 'amount', code: 'INVALID_VALUE', message: 'Must be a positive integer.' }
      ]
    });
  }
  // ...
});
```

## RFC 9457 — what changed in 2023

RFC 9457 supersedes RFC 7807 with minor clarifications:

- `type` defaults to `about:blank` when omitted — the status code alone describes the problem.
- Explicitly discourages using `type` as a plain string code rather than a URI.
- Adds guidance on extension members and `application/problem+xml`.

In practice, differences are minor. Use RFC 9457's guidance but don't worry about which RFC number you cite.

## Error type URI taxonomy

`type` URIs should be stable, hierarchical, and ideally dereferenceable:

```
https://api.example.com/errors/
├── validation-failed          422
├── not-found                  404
├── forbidden                  403
├── unauthenticated            401
├── conflict                   409
├── rate-limit-exceeded        429
└── internal-error             500
```

Clients can match on the full URI for specific handling or on a prefix for category handling:

```typescript
if (error.type === 'https://api.example.com/errors/validation-failed') {
  showFieldErrors(error.errors);
} else if (error.type?.startsWith('https://api.example.com/errors/')) {
  showGenericError(error.title);
}
```

## Common mistakes

> [!warning] Anti-patterns
> - **Changing `title` between occurrences.** It must be stable per `type` — clients key on it.
> - **Putting occurrence-specific detail in `title`.** That belongs in `detail`.
> - **Using a string code instead of a URI for `type`.** `"VALIDATION_ERROR"` is not a URI. Use both if you want readability: `type` as URI + a `code` string extension.
> - **No `instance`.** Without it, clients can't reference a specific error when filing support tickets.
> - **Not setting `Content-Type: application/problem+json`.** Some clients and middleware behave differently on this content type.
> - **No field-level `errors` for validation responses.** `"Validation failed"` without field detail forces clients to show users a useless message.

## References

- RFC 9457 (current): https://datatracker.ietf.org/doc/html/rfc9457
- RFC 7807 (original): https://datatracker.ietf.org/doc/html/rfc7807
- Zalando's error response guidelines: https://opensource.zalando.com/restful-api-guidelines/#176

## Related

- [[Error response formats]]
- [[GraphQL error format]]
- [[API error design guidelines]]
- [[API Documentation]]
- [[OpenAPI Specification]]
