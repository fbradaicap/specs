---
repo: compute-js-apiclarity
spec_type: non_functional
commit: ec477c66d41584893a6c511130717e3b262d0b20
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 17babbb596d66b4f9491ef1a5c7159c424601496251836ebe5a3aebc5a0a5872
generated_at: 2026-06-30T14:54:24.588644386+02:00
generator: specsync
---

## Performance

- The service runs as a **Fastly Compute@Edge WebAssembly module** compiled via `js-compute-runtime`. Execution occurs at Fastly edge PoPs, inheriting edge-network latency characteristics (typically sub-10 ms overhead for compute logic at PoP, excluding upstream calls).
- The primary workload is: query Fastly NGWAF sampled logs API, transform the response, and forward to a locally running APIClarity instance. End-to-end latency is dominated by the two outbound HTTP calls (NGWAF API + APIClarity ingest endpoint).
- No explicit timeout configuration, connection pool sizing, or caching directives are present in the repository.
- No request queue depth, concurrency limits, or throughput targets are specified.

_Not determinable from code_ beyond the above structural observations.

## Scalability

- The service is stateless by construction: it is a Fastly Compute@Edge WASM binary with no local state or database. Each invocation is fully isolated.
- Horizontal scaling is implicit: Fastly's edge platform distributes requests across PoPs and workers automatically. No replica count or autoscaling configuration is managed by this repository.
- The service is designed as a **developer/demo integration tool** (local `make demo` workflow, `kubectl`, local Docker), not a production-scale deployment. No partitioning or sharding is applicable.

_Not determinable from code_ beyond the above.

## Security

- **Credentials**: The following secrets are required at runtime via environment variables: `NGWAF_EMAIL`, `NGWAF_TOKEN`, `NGWAF_CORP`, `NGWAF_SITE`, and a `TRACE-SOURCE-TOKEN` obtained from APIClarity. No secrets-management system (e.g., Vault, Kubernetes Secrets) is specified; responsibility is left to the operator.
- **AuthN toward NGWAF API**: HTTP Basic / Token authentication using `NGWAF_EMAIL` + `NGWAF_TOKEN` passed as request headers (per README usage pattern).
- **Transport toward APIClarity**: The README demo uses `--insecure` with `https://localhost:8443`, indicating TLS is in use but **certificate validation is explicitly disabled** for the local setup (target: this should be hardened for any non-local deployment).
- **AuthZ / input validation**: No middleware-level authZ is evident. The `@fastly/expressly` router handles routing but no authentication middleware for inbound requests to the Compute service is visible in the repository.
- Inbound requests to `/get_sampled_logs` carry credentials as HTTP headers (`NGWAF_EMAIL`, `NGWAF_TOKEN`, etc.), meaning **caller credentials transit over the wire in headers** — transport security of the Fastly Compute listener (`http://127.0.0.1:7676` in local mode) is unencrypted for the demo setup.
- No CSP, rate-limiting, or CORS configuration is evident.

## Observability

- No structured logging framework, metrics exporter, or distributed tracing instrumentation is configured in the repository.
- No `/health` or `/ready` endpoints are detected.
- Observability at the platform level may be available through Fastly's built-in edge logging pipelines, but no log sink or real-time log endpoint is configured in this repository.

_Not determinable from code_ beyond the above.

## Reliability

- No retry logic, circuit breaker, or exponential back-off configuration is evident in the repository.
- No idempotency guarantees are documented; re-sending sampled log data to APIClarity on retry could result in duplicate trace ingestion.
- The service has no persistent state or queue, so a failure mid-flight (NGWAF query succeeds, APIClarity ingest fails) results in silent data loss for that batch with no dead-letter or compensating mechanism visible.
- The dependency on a **locally running APIClarity instance** (accessed via `localhost`) means the service has no availability outside of the developer's local environment as configured; there is no failover target.
- No SLA, RTO, or RPO targets are specified.
