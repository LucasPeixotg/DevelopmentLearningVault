---
tags:
  - infrastructure
  - networking
  - kubernetes
  - microservices
created: 2026-06-03
---
# Service mesh

> Summary: A layer that sits between services and handles cross-cutting concerns — [[Mutual TLS|mTLS]], retries, timeouts, traffic routing, observability, and identity-based authorization — so application code doesn't have to. Traditionally implemented as **sidecar proxies** (one extra container per pod), increasingly as **sidecarless** data planes that share proxies at the node level. The thing that makes [[Mutual TLS|mTLS]] practical at scale.

## What it is

Without a mesh, every service has to handle TLS certs, retries, metrics, service-to-service auth, and load balancing in application code or libraries. With a mesh, those concerns move into a separate layer:

```
Without a mesh:
┌──────────────┐    plain HTTP or self-managed TLS    ┌──────────────┐
│  Service A   │ ───────────────────────────────────► │  Service B   │
└──────────────┘                                       └──────────────┘
   (handles mTLS, retries, metrics, auth — all in app code)

With a mesh (sidecar model):
┌──────────────┬─────────┐         mTLS         ┌─────────┬──────────────┐
│  Service A   │  Proxy  │ ◄──────────────────► │  Proxy  │  Service B   │
└──────────────┴─────────┘                      └─────────┴──────────────┘
       ↑                                                       ↑
   localhost                                              localhost
```

Service A talks plain HTTP to its local proxy. The proxy handles mTLS with B's proxy. B's proxy delivers plain HTTP to Service B. Neither service knows the connection between them is encrypted, authenticated, retried, or observed.

A **control plane** distributes config, certs, and policies to every proxy. Policies like "Service A may call `/v1/charges` on Service B" are written as YAML and enforced by the proxies.

## What it gives you

- **Automatic [[Mutual TLS|mTLS]] between every service.** Each proxy gets a short-lived cert (typically rotated every ~24h) from the mesh's built-in CA. Cert issuance, rotation, and revocation are handled for you. This is the main reason mTLS is *practical* at scale.
- **Identity-based authorization.** Identity comes from the cert (often a SPIFFE ID). Policies say "this identity can call that endpoint."
- **Traffic management.** Canary deploys, blue/green, retries, circuit breaking, fault injection — all configured at the mesh, not in each app.
- **Uniform observability.** Every inter-service request is logged, traced (via OpenTelemetry), and measured with consistent labels.

## Sidecar vs. sidecarless (ambient)

The original mesh model injects a proxy container into every pod. It works but is expensive — every pod runs an extra Envoy process consuming CPU and memory.

**Sidecarless** ("ambient") moves the data plane out of the pod. Istio's ambient mode reached GA in Istio 1.24 (November 2024). Instead of a per-pod proxy, a per-node proxy (`ztunnel`) handles L4 + mTLS, with optional L7 "waypoint" proxies invoked only when needed.

| | Sidecar | Sidecarless / Ambient |
|---|---|---|
| Proxy location | Inside each pod | Per-node + per-namespace waypoints |
| Overhead | Higher (one Envoy per pod) | Lower (shared proxies) |
| Maturity | Battle-tested since 2017 | GA in late 2024; production-ready |
| Migration | App restart per pod | Namespace label change |

Reports of ~45% container reduction after switching from sidecar to ambient are common.

## When to use

- **Many services that need to talk to each other** (rough threshold: 20+).
- **You want [[Mutual TLS|mTLS]] everywhere** without doing cert lifecycle by hand.
- **Multi-team Kubernetes platforms** where each app team shouldn't have to reinvent retries/timeouts/auth.
- **Regulated environments** that need zero-trust networking and uniform audit logs.

## When not to use

- **Small systems** (handful of services). The operational cost isn't worth it.
- **Latency-critical paths** where you can't afford the extra hop (HFT, real-time gaming). Sidecarless modes reduce but don't eliminate this.
- **Non-Kubernetes environments**, though [[#Implementations|Consul Connect]] works on VMs.

## Implementations

**Istio** (sidecar + ambient modes)
- *Why use:* most features, biggest community, ambient mode dramatically reduces overhead.
- *Why not:* steepest learning curve; sidecar mode is still resource-heavy.
- *Handles automatically:* mTLS, cert rotation, traffic policy, observability, mTLS-based authorization.
- *Be aware of:* picking sidecar vs. ambient up front; the control plane is complex; debugging requires understanding Envoy.

**Linkerd**
- *Why use:* simpler, lighter, written for Kubernetes from the start. Often the recommendation for teams that want a mesh without operational pain.
- *Why not:* fewer features than Istio (intentionally).
- *Handles automatically:* mTLS, retries, timeouts, observability.
- *Be aware of:* less flexible policy language; smaller ecosystem of extensions.

**Consul Connect** (HashiCorp)
- *Why use:* works outside Kubernetes — VMs, bare metal, multi-cloud. Integrates with the broader Consul/Vault stack.
- *Why not:* Kubernetes-native options have moved faster in recent years.
- *Handles automatically:* service discovery, mTLS, intentions-based policy.
- *Be aware of:* operating a Consul cluster is its own undertaking.

**Cilium Service Mesh**
- *Why use:* sidecarless from the start (uses eBPF at the kernel level), good performance.
- *Why not:* newer; some L7 features still maturing; requires a compatible kernel.
- *Handles automatically:* mTLS, network policies, observability.
- *Be aware of:* tight coupling with Cilium CNI; eBPF kernel requirements.

**Cloud-managed**
- **Google Cloud Service Mesh** (managed Istio) — production-ready, hands-off.
- **Amazon ECS Service Connect** — AWS's lightweight alternative; the recommended path off App Mesh for ECS users.
- **Amazon VPC Lattice** — AWS's application networking layer; the recommended path off App Mesh for EKS users.
- **AWS App Mesh** — **being deprecated September 30, 2026.** New customers were locked out in September 2024. Migrate to Service Connect, VPC Lattice, or Istio.

## Common mistakes

> [!warning] Anti-patterns
> - **Adopting too early.** Below ~20 services, the mesh's operational cost outweighs its benefits.
> - **Treating it as a black box.** When something breaks, you'll be reading Envoy access logs and the control plane's reconciliation loop. Budget the learning time.
> - **Running mTLS in "permissive" mode in production.** Permissive mode allows non-mTLS traffic so you can migrate gradually. It's an anti-pattern to leave it on permanently — every plaintext path is unauthenticated.
> - **Mixing meshes.** Running Istio and Linkerd in the same cluster, or layering Cilium service mesh on top of Istio. Pick one.
> - **Confusing service identity with user identity.** The mesh authenticates *services* (workloads). It says nothing about which *user* is making a request. You still need [[OAuth 2.0]] or [[JSON Web Tokens|JWTs]] at the application layer for end-user authorization.
> - **Not budgeting for the control plane.** It needs HA, resource limits, and an upgrade strategy of its own.
> - **Skipping the ambient option** if you're starting fresh with Istio in 2026+. The sidecar model is no longer the default recommendation.

## References

- Istio Ambient Mode GA announcement: https://istio.io/latest/blog/2024/ambient-reaches-ga/
- Linkerd: https://linkerd.io/
- AWS App Mesh deprecation notice: https://aws.amazon.com/blogs/containers/migrating-from-aws-app-mesh-to-amazon-ecs-service-connect/
- SPIFFE workload identity: https://spiffe.io/
- *The Service Mesh: What Every Software Engineer Needs to Know* (William Morgan, free essay): https://buoyant.io/service-mesh-manifesto

## Related

- [[Mutual TLS]]
- [[OAuth 2.0]]
- [[JSON Web Tokens]]
- [[Authentication strategies compared]]
- [[Kubernetes]]
- [[Observability]]