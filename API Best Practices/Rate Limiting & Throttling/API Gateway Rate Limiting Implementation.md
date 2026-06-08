---
tags:
  - api-design
  - infrastructure
  - rate-limiting
  - implementation
created: 2026-06-03
implemented: false
---
# API Gateway Rate Limiting Implementation

> Summary: Move rate limiting entirely out of the application and into the infrastructure layer — an API gateway, reverse proxy, or CDN. Rate limiting happens before your Express app is even invoked, so rejected requests never consume application resources. Zero application code required. The right choice at scale and the best complement to application-level limiting. The trade-off: configuration lives outside your codebase, making testing and custom logic harder.

## How it works

```
Client → [Gateway: rate limiting, auth, routing] → Your Express app
                    ↑
          Rejected here — app never sees the request
```

Your Express app needs no rate limiting code at all:

```typescript
// app.ts — rate limiting is the gateway's responsibility
const app = express();
app.use(express.json());
app.post('/v1/charges', chargesHandler);
```

## Common gateway options

### AWS API Gateway

Define usage plans and API keys in the AWS console or via Terraform/CDK:

```yaml
# AWS CDK example
const api = new apigateway.RestApi(this, 'Api');

const plan = api.addUsagePlan('UsagePlan', {
  name: 'Standard',
  throttle: {
    rateLimit: 100,   // requests per second (token bucket rate)
    burstLimit: 200,  // token bucket capacity
  },
  quota: {
    limit: 10000,
    period: apigateway.Period.DAY,
  },
});
```

- Algorithm: token bucket.
- Per-API-key limits via usage plans.
- Managed by AWS — no Redis to operate.

### Kong

The `rate-limiting` plugin, configured per-service or per-route:

```yaml
# kong.yaml (declarative config)
plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 1000
      policy: redis         # use Redis for multi-node accuracy
      redis_host: redis
      redis_port: 6379
      limit_by: consumer    # or ip, credential, service
      error_code: 429
      error_message: "Rate limit exceeded"
```

- Supports fixed window, sliding window, token bucket depending on version.
- Handles per-consumer, per-IP, per-service limits via config.
- Kong runs as a sidecar or standalone; no app changes needed.

### Nginx

Leaky bucket at the reverse-proxy layer — protects the origin from bursts:

```nginx
# nginx.conf
http {
  # Define a zone keyed on binary IP, 10MB of shared memory
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;

  server {
    location /v1/ {
      limit_req zone=api burst=20 nodelay;
      limit_req_status 429;
      proxy_pass http://express_app;
    }
  }
}
```

- Algorithm: leaky bucket.
- Operates before your app processes anything.
- No Redis needed — shared memory zone across workers.

### Cloudflare

Rate limiting rules configured in the dashboard or via Terraform:

```hcl
# Terraform example
resource "cloudflare_rate_limit" "api" {
  zone_id   = var.zone_id
  threshold = 100
  period    = 60

  match {
    request { url_pattern = "api.example.com/v1/*" }
  }

  action {
    mode    = "ban"
    timeout = 30
  }
}
```

- Enforced at Cloudflare's edge — traffic never reaches your origin.
- Supports custom response pages, IP-based, and token-based limiting.
- Best for DDoS protection and unauthenticated traffic.

## Pros and cons

**Pros:**
- **Rate limiting happens before your app runs.** Rejected requests cost nothing in application compute.
- **No Redis dependency in your application.**
- **Gateway enforces limits even if your app crashes or redeploys.**
- Per-IP, per-key, per-plan limits via config — no code.
- Cloudflare-level enforcement stops traffic at the edge, before it hits your infrastructure at all.

**Cons:**
- **Vendor or infrastructure lock-in.** Limits live in AWS/Kong/Cloudflare config, not in your codebase or git history.
- **Hard to test locally.** You need a local Kong/nginx setup or a mocked gateway; `localhost:3000` behaves differently than production.
- **Custom business logic is awkward.** "Apply different limits for trial vs. paid users, looked up from *your* database" is difficult to express in gateway config.
- **Internal traffic bypasses the gateway.** Services calling each other inside your VPC are unprotected. Pair with application-layer limiting for internal traffic.

> [!tip] Use both layers
> Gateway for coarse-grained protection (DDoS, unauthenticated traffic, infrastructure protection) + [[Redis Libraries Rate Limiting Implementation]] or [[Redis-backed sliding window rate limiting Implementation]] for per-user business logic (plan tiers, per-endpoint cost factors). The two layers are complementary, not competing.

## Common mistakes

> [!warning] Anti-patterns
> - **Gateway only, no application-level limiting.** Internal services and any traffic that bypasses the gateway are unprotected.
> - **No local testing strategy.** If your CI environment doesn't mirror the gateway config, rate limit bugs only appear in production.
> - **Gateway config in a wiki, not in code.** Treat gateway config as infrastructure-as-code (Terraform, CDK, Pulumi) so it's reviewed, versioned, and reproducible.
> - **Relying solely on IP limiting at the gateway.** Shared NAT and IPv6 make IP-based limits unreliable for authenticated users. Use API-key or consumer-based limits where possible.
> - **Not forwarding rate limit headers to clients.** Some gateways strip the `RateLimit-*` headers your app sets. Verify end-to-end that clients actually see them.

## References

- AWS API Gateway usage plans: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html
- Kong rate-limiting plugin: https://docs.konghq.com/hub/kong-inc/rate-limiting/
- Nginx `limit_req` module: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html
- Cloudflare rate limiting: https://developers.cloudflare.com/waf/rate-limiting-rules/

## Related

- [[Rate limiting & throttling]]
- [[In-process Rate Limiting Implementation]]
- [[Redis-backed sliding window rate limiting Implementation]]
- [[Redis Libraries Rate Limiting Implementation]]
- [[Rate Limiting in Node and Typescript Comparison]]
- [[Service mesh]]
