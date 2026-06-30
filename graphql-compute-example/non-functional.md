---
repo: graphql-compute-example
spec_type: non_functional
commit: acd3472b4ad7c13c9e26622246735fa9ff16eb3f
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 840b244c7e882e1a2625320741063dbcea7474fb6eb0c568a8f321b43f1df774
generated_at: 2026-06-30T14:49:39.187192395+02:00
generator: specsync
---

## Performance

The service is compiled to WebAssembly (`bin/main.wasm`) and deployed on Fastly's Compute platform, which executes each request as an isolated WASM instance at a Fastly edge PoP. This architecture delivers inherently low-latency request processing by running compute logic geographically close to end users.

- **Latency model:** Edge-side execution minimises round-trip time to the processing layer. GraphQL query resolution against origin data sources is subject to upstream fetch latency (target: minimised by edge caching of origin responses, as described in the README).
- **Caching:** The README explicitly identifies edge caching of origin requests as a primary performance motivation. Specific cache TTLs or `Cache-Control` policies are not determinable from code.
- **Connection/thread pools:** Not applicable; Fastly Compute's WASM runtime handles concurrency at the platform level. No application-level thread or connection pool configuration is present.
- **Timeouts:** No explicit upstream fetch timeouts are configured in the visible source. _Not determinable from code._
- **Bundle optimisation:** Webpack (`^5.89.0`) is used as a pre-build step to bundle and optimise the JavaScript before WASM compilation, reducing binary size and cold-start overhead.

---

## Scalability

- **Horizontal scaling:** Inherent to the Fastly Compute platform. The service is deployed globally across Fastly's network of PoPs; scaling is managed entirely by the platform with no application-level replica or autoscaling configuration required.
- **Statelessness:** The WASM execution model enforces strict statelessness — each request runs in an isolated WASM instance with no shared in-process state. The service is fully stateless by design.
- **Vertical scaling:** Constrained by Fastly Compute's per-request resource limits (CPU time, memory). No custom resource limit configuration is present in the repository.
- **Partitioning/sharding:** Not applicable at the application layer; geographic distribution is handled by Fastly's anycast routing.
- **Replica counts / autoscaling config:** _Not determinable from code_ — managed entirely by the Fastly platform.

---

## Security

- **Transport security:** All traffic is handled at Fastly edge nodes, which terminate TLS. No in-application TLS configuration is present; TLS enforcement is a platform-level concern.
- **AuthN/AuthZ:** _Not determinable from code._ No authentication or authorisation middleware is visible in the dependency list or source structure.
- **Input validation:** GraphQL query parsing and validation is delegated to `graphql-helix` (`^1.13.0`), which performs schema-based validation of incoming GraphQL documents before execution. Malformed or schema-invalid queries are rejected at the parsing/validation stage.
- **Secrets handling:** No secrets, API keys, or credential references are visible in the repository. Fastly Compute supports edge dictionary / config store primitives for secret injection; use of these is _not determinable from code_.
- **Security middleware:** No explicit HTTP security headers middleware (e.g., CORS, CSP, rate-limiting) is evident in the dependencies or configuration.
- **Dependency surface:** Runtime dependencies are minimal (`@fastly/expressly`, `@fastly/js-compute`, `graphql-helix`), reducing the attack surface.

---

## Observability

- **Logging:** `@fastly/js-compute` provides access to Fastly's real-time logging endpoints (e.g., syslog, S3, BigQuery). No specific logging sink or structured logging configuration is present in the repository.
- **Metrics:** _Not determinable from code._ Platform-level metrics (request rate, error rate, cache hit ratio) are available via the Fastly dashboard and real-time stats API, but no application-instrumented metrics (e.g., Prometheus, StatsD) are configured.
- **Tracing:** _Not determinable from code._ No distributed tracing library or configuration (e.g., OpenTelemetry) is present in the dependencies.
- **Health/readiness endpoints:** None detected. Fastly Compute does not use conventional Kubernetes-style health probes; platform-level health monitoring is handled by Fastly infrastructure.
- **Error reporting:** _Not determinable from code._

---

## Reliability

- **Resilience patterns:** _Not determinable from code._ No retry logic, circuit breaker libraries, or bulkhead patterns are visible in the dependency list or build configuration.
- **Idempotency:** GraphQL queries are inherently read-only/idempotent for `query` operations. Mutation support is not evidenced in the README examples, so idempotency concerns are limited.
- **Failure handling:** GraphQL error handling is partially managed by `graphql-helix`, which returns structured GraphQL error responses for execution failures rather than unhandled exceptions. Application-level fallback or graceful degradation logic is _not determinable from code_.
- **Availability:** High availability is a platform guarantee of Fastly Compute's globally distributed edge network. No SLA targets are stated in the repository.
- **Cold starts:** The WASM compilation pipeline (`webpack` → `js-compute-runtime`) produces a pre-compiled binary, which reduces initialisation overhead relative to interpreted execution, supporting low-latency cold-start behaviour (target).
- **Upstream origin resilience:** The README notes that edge caching reduces origin load and improves resilience to origin slowness, but no explicit circuit-breaking or fallback for origin fetch failures is configured in the visible code.
