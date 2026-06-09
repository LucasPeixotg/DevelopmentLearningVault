---
tags: [api-design, security, http, browser, best-practices]
created: 2026-06-03
---

# CORS

> Summary: Cross-Origin Resource Sharing controls whether a web page on one origin may read responses from a different origin. It relaxes the browser's same-origin policy in a controlled way, via response headers the server sends. The critical thing to understand: **CORS is enforced by the browser, not the server** — it protects your users' browsers, not your API. A `curl` request or backend service ignores it entirely.

## The problem it solves

Browsers enforce the **same-origin policy**: JavaScript on `https://app.example.com` cannot read responses from `https://api.other.com` by default. An origin is scheme + host + port — so `https://example.com`, `http://example.com`, and `https://example.com:8080` are all different origins.

This stops a malicious site you visit from silently calling your bank's API using your logged-in session and reading the response. CORS is how a server **opts in** to allowing specific other origins.

## The key mental model

> [!warning] CORS is not a server-side security control
> The browser enforces it. The server only sends headers describing what's permitted; the browser decides whether to hand the response back to the page. Consequences:
> - A `curl` request, mobile app, or backend service ignores CORS entirely — there's no browser to enforce it.
> - The request usually still **reaches your server** even when CORS "blocks" it. The browser just refuses to expose the response to the JavaScript.
>
> Think of CORS as protecting your *users' browsers*, not your API. Use real authentication and authorization for actual security.

## How it works on the wire

### Simple requests

For a basic `GET`, the browser adds an `Origin` header automatically:

```http
GET /v1/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
```

The server responds with `Access-Control-Allow-Origin` if it permits that origin:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json

{ "data": [...] }
```

If the header is missing or doesn't match, the browser throws a CORS error and the JavaScript never sees the response — even though the server processed the request.

### Preflight requests

For anything beyond a "simple" request — `PUT`/`DELETE`/`PATCH`, a custom header like `Authorization`, or a JSON content type — the browser first sends an `OPTIONS` preflight to check whether the real request is allowed:

```http
OPTIONS /v1/data HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization, content-type
```

The server replies with what it permits:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

Only if the preflight passes does the browser send the actual `DELETE`. `Access-Control-Max-Age` lets the browser cache the preflight result so it doesn't re-check every request.

### Credentials

If the request carries cookies or HTTP auth, the server must explicitly allow it — and **cannot use a wildcard origin**:

```http
Access-Control-Allow-Origin: https://app.example.com   ← must be exact, not *
Access-Control-Allow-Credentials: true
```

## The headers

| Header | Direction | Purpose |
|---|---|---|
| `Origin` | Request | Set by the browser — the requesting origin |
| `Access-Control-Request-Method` | Preflight request | The method the real request will use |
| `Access-Control-Request-Headers` | Preflight request | The headers the real request will send |
| `Access-Control-Allow-Origin` | Response | Which origin is allowed (exact or `*`) |
| `Access-Control-Allow-Methods` | Preflight response | Allowed methods |
| `Access-Control-Allow-Headers` | Preflight response | Allowed request headers |
| `Access-Control-Allow-Credentials` | Response | Whether cookies/auth are allowed (`true`) |
| `Access-Control-Max-Age` | Preflight response | How long to cache the preflight result |
| `Access-Control-Expose-Headers` | Response | Which response headers JS may read (e.g. `RateLimit-*`) |

## Express implementation

Use the `cors` middleware rather than setting headers by hand (see [[Express production middleware baseline]]):

```typescript
import cors from 'cors';

app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'], // allowlist
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Authorization', 'Content-Type'],
  credentials: true,          // allow cookies — requires exact origin, not *
  maxAge: 86400,
  exposedHeaders: ['RateLimit-Limit', 'RateLimit-Remaining'], // let JS read these
}));
```

The middleware handles the `OPTIONS` preflight automatically — without it, every non-simple request fails before the real request is attempted.

## Common mistakes

> [!warning] Anti-patterns
> - **Reflecting the `Origin` header blindly** (`origin: true` or echoing whatever the client sent). This allows every origin and defeats the purpose. Use an allowlist.
> - **`Access-Control-Allow-Origin: *` with credentials.** The browser rejects this combination — wildcard and credentials are mutually exclusive.
> - **Forgetting to handle `OPTIONS`.** If the preflight route isn't handled, every non-simple request fails. The `cors` middleware does this for you.
> - **Treating CORS as API security.** It governs browser-based cross-origin access only. Non-browser clients ignore it. Authenticate and authorise properly regardless.
> - **Not exposing custom response headers.** By default JS can only read a handful of "simple" response headers. To let clients read `RateLimit-*` or pagination headers, list them in `Access-Control-Expose-Headers`.
> - **Debugging a CORS error as if the API is broken.** A CORS error usually means a missing response header, not a failed request. The fix is almost always server-side config, not changing the request.

## References

- MDN CORS guide: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- Fetch standard (CORS protocol): https://fetch.spec.whatwg.org/#http-cors-protocol
- `cors` middleware: https://github.com/expressjs/cors

## Related

- [[Express production middleware baseline]]
- [[Authentication strategies compared]]
- [[API error design guidelines]]
- [[Rate limiting & throttling]]
