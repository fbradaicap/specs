---
repo: compute-sdk-python
spec_type: non_functional
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:33.154103392+02:00
generator: specsync
---

## Performance

- **Request timeout**: Configurable per test class via `REQUEST_TIMEOUT` class attribute; default is **10 seconds**. A custom value of 30 seconds is shown as an example in the README.
- **Execution model**: Applications compile to WebAssembly (WASM) and run inside the Fastly Compute edge runtime (Viceroy locally). The vCPU time in milliseconds is exposed to application code via `compute_runtime.get_vcpu_ms()`, indicating per-request CPU billing granularity.
- **Connection/thread pools**: _Not determinable from code._
- **Caching**: _Not determinable from code._
- **Tuning**: No explicit throughput targets or latency budgets are configured beyond the per-request timeout.

---

## Scalability

- **Execution environment**: The SDK targets the Fastly Compute edge platform, which provides inherently stateless, per-request WASM invocations. Each request is isolated by design; no shared in-process state is expected between requests.
- **Statelessness**: Enforced by the WASM component model — no persistent in-process state survives across requests.
- **Replica counts / autoscaling**: Managed entirely by the Fastly edge platform, not configured in this repository.
- **Horizontal/vertical scaling, partitioning/sharding**: _Not determinable from code._ Scaling is delegated to the Fastly edge network.

---

## Security

- **AuthN/AuthZ**: _Not determinable from code._ No authentication or authorisation middleware is present in the SDK or example applications.
- **Transport security**: TLS termination is handled by the Fastly edge platform upstream of the WASM runtime; no TLS configuration is present in this repository.
- **Secrets handling**: Fastly Config Stores (`ConfigStore` resource class) are available for secrets/configuration injection at the edge; no plaintext secrets are embedded in source or manifests.
- **Input validation**: No explicit input-validation middleware is configured. Example routes accept path parameters (e.g., `/hello/<name>`) without visible sanitisation.
- **Security dependencies / middleware**: None detected beyond what is provided by the Fastly Compute host runtime environment itself.

---

## Observability

- **Logging**: The Rust build tooling uses `log` (v0.4) and `env_logger` (v0.11) for build-time diagnostic output. Runtime log endpoints are exposed to application code via the `LogEndpoint` resource class in the SDK.
- **vCPU metrics**: `compute_runtime.get_vcpu_ms()` is available to application code and is exercised in example `/info` endpoints, providing per-request vCPU time.
- **Viceroy server output**: The pytest plugin (`fastly_compute.pytest_plugin`) automatically surfaces recent Viceroy server logs on test failures, aiding local debugging.
- **Distributed tracing**: _Not determinable from code._
- **Metrics exposition** (Prometheus/StatsD/etc.): _Not determinable from code._
- **Health/readiness endpoints**: No dedicated health or readiness endpoint is defined in the SDK; the example apps expose `/info` for basic runtime status (target for operational use, not a formal health check).

---

## Reliability

- **Retries**: _Not determinable from code._
- **Circuit breakers**: _Not determinable from code._
- **Timeouts**: Per-request HTTP client timeout is enforced in the test harness (default 10 s, configurable). No server-side request timeout configuration is present in the SDK itself.
- **Idempotency**: _Not determinable from code._
- **Error handling**: The SDK provides a structured exception-remapping layer (`remap_wit_errors` decorator) that converts WIT `result`-type errors into typed Python exceptions (`FastlyError` hierarchy, including `UnexpectedFastlyError` as a catch-all). This prevents raw WIT binding exceptions from leaking into application code.
- **Resource lifecycle**: All Fastly host resources (`ConfigStore`, `RateCounter`, `PenaltyBox`, `LogEndpoint`) implement the context-manager protocol via `FastlyResource`; failure to close a resource before request completion will eventually be handled by the garbage collector, though post-close access will result in a host trap.
- **Failure isolation**: WASM execution provides strong per-request isolation; a panic or unhandled exception in one request cannot corrupt state for another.
- **Availability/recovery**: Dependent on Fastly edge platform SLAs; no SDK-level availability configuration is present.
