---
repo: compute-starter-kit-javascript-expressly
spec_type: non_functional
commit: d0945ec18eb2d221a3d3da811ba45d7f46dfd215
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 07c150c73a3ef746e559192e517f0af4480943c215c8af3ad7b45210ea33efec
generated_at: 2026-06-30T14:51:30.820608466+02:00
generator: specsync
---

## Performance

The service runs as a Fastly Compute@Edge application compiled to WebAssembly (`./bin/main.wasm`) and executed on Fastly's globally distributed edge network. Key performance characteristics derive from this platform:

- **Latency**: Requests are handled at the edge node nearest to the client, targeting sub-millisecond cold-start times inherent to the Wasm execution model (target).
- **Throughput**: Governed by Fastly's platform capacity per point of presence rather than any application-level configuration; no explicit rate limits or throughput caps are configured in this repository.
- **Timeouts**: No application-level timeout configuration is present in the code or manifests. Platform-default Fastly Compute request timeouts apply.
- **Connection/thread pools**: _Not determinable from code._ The Wasm runtime model is single-threaded per request; no connection pool configuration is present.
- **Caching**: No caching directives, backend fetch policies, or cache-control headers are configured in the starter kit. Responses are synthetic; no backend is used.
- **Tuning**: No additional tuning (e.g., buffer sizes, worker counts) is evident in the configuration.

## Scalability

- **Horizontal scaling**: Inherent to the Fastly Compute@Edge platform. The compiled Wasm artifact is distributed to and executed across all Fastly PoPs automatically; no explicit replica count or autoscaling configuration is managed by this repository.
- **Statelessness**: The service is stateless by design. No databases, shared state stores, session management, or persistent connections are present. Each request is handled independently within an isolated Wasm instance.
- **Vertical scaling**: Governed entirely by Fastly's platform resource limits per Wasm instance; not configurable within this repository.
- **Partitioning/sharding**: _Not determinable from code._ Not applicable at the application level given the platform model.

## Security

- **AuthN/AuthZ**: _Not determinable from code._ No authentication or authorisation middleware is configured in the starter kit.
- **Transport security**: TLS termination is handled by the Fastly platform before requests reach the Compute worker. No application-level TLS configuration is present.
- **Secrets handling**: No secrets, API keys, or environment variables are referenced in the code or manifests. No backend origins are configured, eliminating credential management concerns in this starter.
- **Input validation**: _Not determinable from code._ No explicit input validation middleware is configured; any validation would be provided by `@fastly/expressly` routing primitives.
- **Security middleware**: No security-specific middleware (e.g., CORS, CSP, rate limiting, request sanitisation) is present in the configuration.
- **Vulnerability reporting**: A `SECURITY.md` is referenced in the README, indicating a documented responsible-disclosure process exists for the owning organisation.

## Observability

- **Logging**: _Not determinable from code._ No logging configuration is present in the starter kit. The Fastly Compute platform provides real-time log streaming to configured log endpoints, but none are configured here.
- **Metrics**: _Not determinable from code._ No application-level metrics instrumentation (e.g., Prometheus, StatsD) is present. Platform-level metrics are available via the Fastly dashboard.
- **Tracing**: _Not determinable from code._ No distributed tracing instrumentation is configured.
- **Health/readiness endpoints**: None detected. The service exposes no dedicated health or readiness routes; the Fastly platform manages service availability directly.

## Reliability

- **Resilience patterns**: _Not determinable from code._ No retries, circuit breakers, bulkheads, or fallback logic are configured. The starter kit generates only synthetic responses and uses no backends, removing the primary surface area for these patterns.
- **Idempotency**: All responses are synthetically generated at the edge with no side effects, making all requests inherently idempotent.
- **Failure handling**: Error handling behaviour is determined by `@fastly/expressly` framework defaults and the Fastly Compute runtime; no custom error handlers are evident in the repository.
- **Availability/recovery**: High availability is a platform-level concern provided by Fastly's globally redundant edge network. No application-level availability configuration (e.g., replica health checks, restart policies) is present in this repository.
