---
tags: [api-design, documentation, best-practices, lifecycle, deprecation]
created: 2026-06-03
---
# API lifecycle communication

> Summary: A changelog page alone doesn't reach most developers. Breaking changes need a layered communication strategy — in-band headers, dashboard banners, targeted emails, SDK warnings, and direct outreach for high-volume consumers. Before any of this is effective, you need instrumentation: you must know exactly who is still calling deprecated endpoints and how often.

## The communication stack

Not every developer reads the changelog. Each layer catches a different type:

```
Layer 1 — In-band (every response to the deprecated endpoint)
    Deprecation: true
    Sunset: Sat, 31 Mar 2026 23:59:59 GMT
    Link: <https://docs.example.com/migration/v1-v2>; rel="sunset"

Layer 2 — Developer portal / dashboard
    Banner: "v1 API sunsets March 31, 2026. Migrate now →"
    Email: sent to every account that called a deprecated
           endpoint in the last 30 days

Layer 3 — Changelog
    Entry with explicit breaking/non-breaking labels
    (see [[API Changelog]])

Layer 4 — Documentation
    Deprecation notice on every affected endpoint's reference page
    Migration guide linked from every notice
    (see [[API Migration guides]])

Layer 5 — SDK
    @deprecated annotation on affected methods (shows in IDE)
    console.warn() on first use in development
    CHANGELOG.md entry in the SDK repo
    (see [[SDK versioning]])

Layer 6 — Direct outreach
    Personalised email to high-volume consumers
    (those making >N calls/day to the deprecated endpoint)
```

> [!tip] In-band headers are the most reliable layer
> `Deprecation` and `Sunset` headers appear in every response. An HTTP client library that surfaces these (or a logging middleware that alerts on them) will catch developers who never read email or docs. Always send headers even if you do everything else too.

## Identifying who is still on deprecated versions

Instrument deprecated endpoints before you communicate — you need to know who to reach and with what urgency:

```typescript
// Middleware that tracks deprecated endpoint usage
app.use('/v1/*', (req, _res, next) => {
  metrics.increment('api.deprecated.request', {
    version:    'v1',
    path:       req.path,
    account_id: req.auth?.accountId,
    key_id:     req.auth?.keyId,
  });
  next();
});
```

From this data:
- **Email all accounts** with deprecated usage in the last 30 days.
- **Prioritise by volume** — an account making 100,000 deprecated calls/day needs a phone call, not an email.
- **Track migration progress** — watch deprecated traffic decline as clients migrate.
- **Set a go/no-go gate** — if 5% of revenue is still on v1 the week before sunset, delay and investigate.

A dashboard showing "X accounts, Y% of traffic still on deprecated API" is the most useful operational tool for managing a sunset.

## Consumers who never upgrade

There is always a long tail. Strategies from gentlest to hardest:

| Strategy | When to use |
|---|---|
| **Extended sunset window** | Traffic still high close to deadline — announce extension, but do it once only |
| **Targeted email to holdouts** | Accounts still calling deprecated endpoint 30 days before sunset |
| **Gradual degradation** | Increase latency, reduce rate limits on deprecated endpoints to create pressure |
| **Hard cutoff (`410 Gone`)** | After all other strategies, on the announced date |

### Targeted email that gets action

```
Subject: Action required: your Acme Corp integration will break on March 31

Your account (Acme Corp) is making approximately 1,200 calls/day to
POST /v1/charges, which will be sunset on March 31, 2026.

After that date, these calls will return 410 Gone.

The migration takes approximately 2 hours:
[single most important action, linked to full guide]

Reply to this email if you need help.
```

Specific account name, specific volume, specific date, specific action. Generic "some of our customers" emails get ignored.

### Gradual degradation

Gradually reduce reliability on the deprecated endpoint before the hard cutoff:

```
T-90 days: Deprecation headers added
T-60 days: Rate limits on deprecated endpoint reduced by 50%
T-30 days: 1% of deprecated requests return 503
T-14 days: 5% of deprecated requests return 503
T-0:       100% return 410 Gone
```

This surfaces real-world breakage before full sunset and creates urgency without a cliff edge.

> [!warning] Only degrade with advance notice
> Gradual degradation must be announced with the same notice period as the sunset itself. Surprise degradation is worse than a hard cutoff.

### The hard cutoff

```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type":   "https://api.example.com/errors/endpoint-sunset",
  "title":  "This endpoint has been sunset.",
  "status": 410,
  "detail": "POST /v1/charges was sunset on 2026-03-31. Use POST /v2/charges.",
  "instance": "https://docs.example.com/migration/v1-v2"
}
```

If you've executed the previous layers well, traffic to the deprecated endpoint should be under 1% by the sunset date. If it's still at 20%, you have a process problem upstream, not a cutoff problem.

## Common mistakes

> [!warning] Anti-patterns
> - **Changelog-only communication.** Not every developer reads it.
> - **Blast emails about changes affecting 5% of integrations.** Target affected accounts only — mass emails train developers to ignore you.
> - **Communicating before instrumenting.** Know who's affected before reaching out.
> - **Extending the sunset deadline more than once.** One extension is forgivable. Two trains clients to ignore your dates.
> - **No go/no-go gate before cutoff.** Decide in advance: what traffic percentage triggers a delay?

## References

- RFC 8594 (Sunset header): https://datatracker.ietf.org/doc/html/rfc8594
- GitHub's deprecation communication approach: https://docs.github.com/en/rest/overview/breaking-changes
- Stripe upgrade guide: https://stripe.com/docs/upgrades

## Related

- [[API Changelog]]
- [[API Migration guides]]
- [[SDK versioning]]
- [[API Deprecation & Sunset]]
- [[Rate limiting & throttling]]
