---
tags: [api-design, error-handling, best-practices]
created: 2026-06-03
---
# Error response formats

> Summary: How you structure error responses determines whether clients can handle errors programmatically or are forced to guess. Three conventions exist: **RFC 7807 Problem Details** (the standard, use this for new REST APIs), **custom error envelopes** (what most APIs built before RFC 7807 use), and **GraphQL errors** (a different model entirely — always `200 OK`, errors live in the body). The format is less important than consistently answering: what happened, why, where, and what to do next.

## The three formats

|                     | [[RFC 7807 Problem Details]] | Custom envelope     | [[GraphQL error format]] |
| ------------------- | ---------------------------- | ------------------- | ------------------------ |
| **Content-Type**    | `application/problem+json`   | `application/json`  | `application/json`       |
| **Status on error** | Correct HTTP status          | Correct HTTP status | Always `200 OK`          |
| **Standardised**    | Yes (RFC 9457)               | No                  | GraphQL spec             |
| **Partial success** | No                           | No                  | Yes                      |
| **Best for**        | REST APIs (new)              | REST APIs (legacy)  | GraphQL APIs             |

## Quick shapes

**RFC 7807:**
```json
{
  "type":     "https://api.example.com/errors/validation-failed",
  "title":    "Validation failed",
  "status":   422,
  "detail":   "The 'amount' field must be a positive integer.",
  "instance": "/v1/charges/req_abc123"
}
```

**Custom envelope (Stripe-style):**
```json
{
  "error": {
    "type":    "invalid_request_error",
    "code":    "amount_too_small",
    "message": "Amount must be at least 50 cents.",
    "param":   "amount"
  }
}
```

**GraphQL:**
```json
{
  "data": { "charge": null },
  "errors": [{
    "message": "Amount must be positive",
    "path": ["charge"],
    "extensions": { "code": "INVALID_VALUE" }
  }]
}
```

## When to use which

> [!tip] Decision rule
> - **New REST API** → [[RFC 7807 Problem Details]]. It's the standard; use it.
> - **Existing REST API with clients in production** → keep your custom envelope. A format change is a breaking change.
> - **GraphQL API** → [[GraphQL error format]]. No choice — it's defined by the spec.
> - **Migrating a legacy API to RFC 7807** → only worth it if you're versioning anyway.

## What every format must answer

Regardless of the format, a good error response answers:

1. **What happened** — machine-readable `type` or `code`
2. **Why it happened** — human-readable `message` or `detail`
3. **Where it happened** — field-level detail for validation errors
4. **What to do next** — retry? fix input? contact support with a `request_id`?

See [[API error design guidelines]] for what to always include and what to never leak.

## References

- RFC 9457 (Problem Details): https://datatracker.ietf.org/doc/html/rfc9457
- GraphQL error spec: https://spec.graphql.org/October2021/#sec-Errors
- Stripe error codes (well-designed custom envelope): https://stripe.com/docs/error-codes

## Related

- [[RFC 7807 Problem Details]]
- [[GraphQL error format]]
- [[API error design guidelines]]
- [[API Documentation]]
- [[Rate limiting & throttling]]
