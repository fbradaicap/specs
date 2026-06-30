---
repo: compute-sdk-python
spec_type: non_functional
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:10.797313958+02:00
generator: specsync
---

## Performance

- **Request timeout**: Configurable per test class via `REQUEST_TIMEOUT` class attribute; default is **10 seconds**. A custom example sets 30 seconds. These govern HTTP client timeouts when exercising the WASM application under Viceroy.
- **Execution model**: Applications compile to WebAssembly (WASM) and run inside the Fastly Compute runtime (Viceroy locally). Each request is handled as an isolated WASM component invocation; there is no persistent in-process thread pool or connection pool managed by this SDK.
- **vCPU time instrumentation**: The example applications expose `compute_runtime.get_vcpu_ms()` (Fastly's per-request vCPU millisecond counter), enabling per-request CPU cost visibility.
- **Caching**: _Not determinable from code._
- **Connection/thread pools**: _Not determinable from code._ (Managed externally by the Fastly Compute platform, not by this SDK.)

---

## Scalability

- **Horizontal scaling**: The SDK targets the Fastly Compute (edge) platform, where scaling is handled entirely by Fastly's infrastructure. Each WASM component invocation is stateless and share-nothing by design.
- **Statelessness**: Enforced by the WASM component model; no shared mutable state between requests is possible within a single component instance.
- **Replica counts / autoscaling**: _Not determinable from code._ These are platform-level concerns outside the SDK.
- **Partitioning/sharding**: _Not determinable from code._
- **Python version floor**: Requires CPython ≥ 3.12; a stable ABI wheel (`abi3-py312`) is emitted so a single build works on CPython 3.12 and all future versions without rebuilding.

---

## Security

- **Transport security**: _Not determinable from code._ TLS termination is handled by the Fastly edge platform before reaching the WASM component.
- **AuthN/AuthZ**: _Not determinable from code._ No authentication or authorization middleware is present in the SDK itself; responsibility lies with application code built on top of the SDK.
- **Secrets handling**: _Not determinable from code._ Fastly Compute supports Config Stores (referenced via `ConfigStore` resource wrapper in `_resource.py`) for secret/configuration injection, but no secrets management policy is codified in this repository.
- **Input validation**: No generic input-validation middleware is present in the SDK. Example applications (Bottle, Flask) rely on framework-level routing validation only.
- **Dependency supply chain**: The Rust workspace pins dependencies via `Cargo.lock` (implied by workspace structure). `componentize-py` is sourced from a specific git tag (`v0.23.0`) of the Bytecode Alliance repository, reducing supply-chain drift risk.
- **WIT error isolation**: The `remap_wit_errors` decorator wraps WIT-generated `Err` values into typed Python exceptions (`FastlyError` hierarchy), preventing raw WIT internals from leaking to application code. Unmapped error types are wrapped in `UnexpectedFastlyError` rather than being swallowed or re-raised opaquely.
- **WASM sandbox**: All application code runs inside a WASM component sandbox enforced by the Fastly runtime; host capabilities are only accessible through explicitly declared WIT interfaces.

---

## Observability

- **Logging (host)**: `env_logger` (Rust) and `log` crate are included in the build toolchain crate (`fastly-compute-py`), enabling structured log emission during SDK build/tooling execution. Log level is configurable via the `RUST_LOG` environment variable.
- **vCPU metrics**: Applications can read `compute_runtime.get_vcpu_ms()` to observe per-request CPU time and include it in responses or logs.
- **Test failure diagnostics**: The `fastly_compute.pytest_plugin` plugin automatically surfaces recent Viceroy server stdout/stderr output on test failures, providing runtime-level observability during CI.
- **Health/readiness endpoints**: _Not determinable from code._ No dedicated health or readiness endpoint is defined within the SDK; example apps expose `/info` which returns runtime metadata and can serve as a functional smoke-check.
- **Distributed tracing**: _Not determinable from code._
- **Metrics export**: _Not determinable from code._ Platform-level metrics are provided by Fastly's observability infrastructure, not by this SDK.

---

## Reliability

- **Error handling — intentional fault injection**: Example apps implement a `/error` endpoint that raises `RuntimeError` to verify that the WSGI/WIT adapter correctly propagates unhandled exceptions to the Fastly runtime without crashing the test harness.
- **WIT error remapping**: The `remap_wit_errors` decorator provides a structured error-normalisation layer so that WIT `result`-type errors are always surfaced as typed Python exceptions. This prevents silent swallowing of host-layer errors and ensures callers can catch the `FastlyError` base class for broad resilience handling.
- **Resource lifecycle management**: `FastlyResource` / `WitResource` enforce explicit resource cleanup via context managers and `close()`. Post-close usage results in a deterministic trap rather than undefined behaviour, improving failure detectability.
- **Retries / circuit breakers**: _Not determinable from code._ These patterns are not implemented in the SDK; they are the responsibility of application code or the Fastly platform.
- **Idempotency**: _Not determinable from code._
- **Dynamic port allocation in tests**: The Viceroy test harness uses dynamic port allocation to prevent port-collision failures during parallel or repeated test runs, improving CI reliability.
- **Availability/recovery (platform)**: WASM component isolation means a crash in one request invocation cannot affect other concurrent invocations; recovery is handled by the Fastly Compute runtime.
