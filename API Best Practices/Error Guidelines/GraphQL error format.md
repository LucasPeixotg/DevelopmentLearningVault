---
tags: [api-design, error-handling, graphql]
created: 2026-06-03
---

# GraphQL error format

> Summary: GraphQL always returns `200 OK` — even on errors. Error information lives in the response body alongside the data, in a top-level `errors` array. The most important difference from REST: **partial success**. Some fields can resolve successfully while others fail, and the client receives both. Machine-readable error classification goes in `extensions`, which the spec leaves open-ended.

## Structure

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": {
    "user": { "id": "42", "name": "Alice" },
    "orders": null
  },
  "errors": [
    {
      "message":   "Insufficient permissions to read orders",
      "locations": [{ "line": 4, "column": 3 }],
      "path":      ["orders"],
      "extensions": {
        "code": "FORBIDDEN",
        "type": "https://api.example.com/errors/forbidden"
      }
    }
  ]
}
```

`user` resolved successfully. `orders` failed. The client gets both.

## Fields

| Field | Required | Purpose |
|---|---|---|
| `message` | Yes (spec) | Human-readable description. |
| `locations` | Optional | Which line/column in the query caused the error. |
| `path` | Optional | Which field in the response is null due to this error. |
| `extensions` | Optional | Custom metadata — `code`, `type`, HTTP-equivalent status, field-level detail. |

## `extensions` — where the useful stuff goes

The GraphQL spec deliberately leaves `extensions` open. Put your error classification here:

```json
"extensions": {
  "code":   "VALIDATION_ERROR",
  "type":   "https://api.example.com/errors/validation-failed",
  "status": 422,
  "fields": [
    { "path": "input.amount",   "message": "Must be positive" },
    { "path": "input.currency", "message": "Must be ISO 4217" }
  ]
}
```

Common `code` conventions:
- `NOT_FOUND` — resource doesn't exist
- `FORBIDDEN` — authenticated but not authorised
- `UNAUTHENTICATED` — no valid credentials
- `VALIDATION_ERROR` — input failed validation
- `INTERNAL_ERROR` — catch-all for unexpected errors

These mirror gRPC status codes and are widely adopted in the GraphQL ecosystem (Apollo Server uses them by default).

## Partial success — the key difference from REST

REST is all-or-nothing per request. GraphQL can succeed on some fields and fail on others within a single response. Whether a field error propagates up to its parent depends on nullability in the schema.

```graphql
# If orders is non-nullable, an error on orders nulls out the parent
type User {
  id: ID!
  orders: [Order!]!   # non-nullable — error propagates up
}

# If orders is nullable, the field becomes null and data.user survives
type User {
  id: ID!
  orders: [Order]     # nullable — error stays local
}
```

> [!tip] Design nullable for optional relationships
> Make fields nullable if you want errors to stay local and allow partial success. Non-nullable fields that fail propagate the null up the tree — which can wipe out an entire successful response.

## When `data` is null entirely

If the error is at the root level (e.g. an unauthenticated request), `data` may be `null`:

```json
{
  "data": null,
  "errors": [
    {
      "message": "Not authenticated",
      "extensions": { "code": "UNAUTHENTICATED" }
    }
  ]
}
```

Some clients treat `data: null` + `errors` as a "hard failure" vs. `data` present + `errors` as a "partial failure". Design your client error handling to distinguish these.

## Common mistakes

> [!warning] Anti-patterns
> - **Returning HTTP `4xx`/`5xx` for GraphQL errors.** Breaks clients that expect `200 OK` and handle errors via the body. Only use non-200 for transport-level failures (malformed JSON, server crash before any response).
> - **Putting sensitive detail in `message`.** `message` goes to clients. Stack traces, DB errors, internal paths — keep them server-side.
> - **No `extensions.code`.** `message` alone is not machine-readable. Always include a `code` in `extensions` so clients can switch on error type.
> - **Inconsistent `code` values across resolvers.** Define an enum of allowed codes and enforce it.
> - **Non-nullable everything.** A single resolver failure can null out the entire response. Make relationships nullable unless you're certain they'll always resolve.
> - **Not logging the full error server-side.** You return a safe message to clients, but you need the full detail internally. Use a correlation ID in `extensions` to link them.

## References

- GraphQL error spec: https://spec.graphql.org/October2021/#sec-Errors
- Apollo Server error handling: https://www.apollographql.com/docs/apollo-server/data/errors/
- GraphQL error codes convention: https://www.apollographql.com/docs/apollo-server/data/errors/#built-in-error-codes

## Related

- [[Error response formats]]
- [[RFC 7807 Problem Details]]
- [[API error design guidelines]]
