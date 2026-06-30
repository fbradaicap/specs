---
repo: Viceroy
spec_type: non_functional
commit: d7ccf104ed0b5c4399a97a9442279c97705d749a
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ed522038d502957e706c70229783a2ab1afd79135845f526027725ea31a6bed8
generated_at: 2026-06-30T14:52:45.787430157+02:00
generator: specsync
---

## Performance

Viceroy is a **local development and testing tool** (a CLI daemon) for Fastly Compute, not a production service. Performance characteristics are derived from its Rust/Wasmtime-based execution model and Tokio async runtime configuration.

- **Async runtime:** Uses `tokio` v1.49.0 with the `full` feature set, providing a multi-threaded async executor. No explicit worker-thread count or Tokio runtime builder configuration is observable from the snapshot.
- **HTTP stack:** Uses `hyper` v0.14.26 (full features) for HTTP/1 and HTTP/2 handling. No explicit connection pool sizes, keep-alive tuning, or request queue limits are observable.
- **TLS:** `rustls` v0.21 with `tokio-rustls` v0.24.1 for TLS termination; no session cache sizing is observable.
- **Caching:** `moka` v0.12.12 (`future`-enabled async cache) is included as a dependency in `viceroy-lib`, but specific cache sizes, TTLs, or eviction policies are not determinable from the snapshot.
- **WebAssembly execution:** Wasmtime v39.0.2 (with `call-hook` feature) is used as the Wasm engine. No explicit fuel limits, epoch interruption intervals, or memory limits are observable from the snapshot.
- **Build optimisation:** A `release-lto-stripped` profile applies fat LTO, single codegen unit, and symbol stripping for release binaries, yielding maximum runtime performance at the cost of build time. Development builds use `opt-level = 1` to keep integration-test compilation times reasonable.
- Explicit latency SLAs, throughput targets, or connection-pool sizes: _Not determinable from code._

## Scalability

- **Deployment model:** Viceroy is a single-process CLI tool intended for local developer use, not a horizontally scaled microservice. No replica counts, autoscaling configuration, Kubernetes manifests, or load-balancer configuration are present.
- **Statelessness:** Each Wasm guest instance is isolated within Wasmtime; no shared in-process mutable state is implied across requests beyond what `moka` caches. The process itself is inherently single-node.
- **Vertical scaling:** Governed by the host machine's CPU and memory. Tokio's multi-threaded executor will use all available CPU cores by default.
- **Horizontal scaling / partitioning / sharding:** _Not determinable from code._ (Not applicable to a local testing tool.)

## Security

- **Transport security:** TLS is supported via `rustls` v0.21 (`dangerous_configuration` feature enabled in the workspace) and `tokio-rustls` v0.24.1. `rustls-native-certs` v0.6.3 is used for loading platform certificate stores. The `dangerous_configuration` feature is present in the workspace definition, indicating that certificate verification can be overridden (intended for local testing contexts).
- **Secrets handling:** No secret management integration (Vault, AWS Secrets Manager, etc.) is present. TLS private keys are loaded via `rustls-pemfile`; no encrypted-at-rest key handling is observable.
- **AuthN/AuthZ:** _Not determinable from code._ No authentication or authorisation middleware is evident; as a local development tool this is expected.
- **Input validation:** `regex` v1.3.9 and `url` v2.3.1 are present for input parsing; specific validation rules are not determinable from the snapshot alone.
- **Wasm sandboxing:** Guest Wasm programs execute inside Wasmtime's sandboxed environment, providing memory isolation and capability-based access control as a security boundary.
- **Container base image:** The Dockerfile uses `ubuntu:bionic` (18.04), which is end-of-life. This is a build/CI image, not a production runtime image.
- **Supply chain:** The `call-hook` feature of Wasmtime is enabled, allowing the host to intercept all host calls; this can be used to enforce security policies on guest behaviour.

## Observability

- **Logging/Tracing:** `tracing` v0.1.37 and `tracing-futures` v0.2.5 are used throughout `viceroy-lib`. The CLI (`viceroy`) adds `tracing-subscriber` v0.3 with `env-filter` and `fmt` features, enabling structured, filterable log output controlled via the `RUST_LOG` environment variable (standard `tracing-subscriber` behaviour).
- **Metrics:** _Not determinable from code._ No Prometheus, OpenTelemetry metrics exporter, or StatsD client is present.
- **Distributed tracing:** No OpenTelemetry trace exporter or Jaeger/Zipkin integration is present beyond the `tracing` instrumentation library itself.
- **Health/readiness endpoints:** _Not determinable from code._ No HTTP health-check routes are detected; not applicable for a CLI tool.

## Reliability

- **Resilience patterns:** _Not determinable from code._ No circuit-breaker library (e.g., `failsafe`, `tower`) or retry middleware is present.
- **Wasm fault isolation:** Each guest Wasm module runs in a Wasmtime instance. Fatal host-call errors result in instance termination (the `test-fatalerror-config` feature and the `trap-test` integration test explicitly exercise this path), preventing a faulting guest from affecting the host process.
- **Idempotency:** _Not determinable from code._
- **Timeouts:** No explicit request-level or upstream-call timeout configuration is observable in the snapshot.
- **Availability/recovery:** As a local CLI tool, high-availability concerns (redundancy, leader election, graceful restart) are not applicable. The tool is expected to be restarted manually by the developer.
- **CI reliability:** GitHub Actions concurrency control (`cancel-in-progress: true`) is configured per workflow, preventing duplicate CI runs from consuming resources simultaneously.
