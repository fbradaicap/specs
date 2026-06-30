---
repo: rezolus
spec_type: non_functional
commit: f8beb6fce7394ff5421f0ad9371adf2bd25903d7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 46fbc1f0c5073dc355a6ecb735431d5c6d3a5672528bcd134a89f460a7891e0e
generated_at: 2026-06-30T14:54:33.524586697+02:00
generator: specsync
---

## Performance

Rezolus is purpose-built as a **low-overhead, high-resolution telemetry agent**. Its core design goal is minimal perturbation of the system being observed, achieved through eBPF-based instrumentation.

- **eBPF instrumentation**: Metric collection is performed in-kernel via eBPF programs (`libbpf-rs`, `libbpf-sys`), eliminating costly user/kernel context switches for each observation and keeping collection overhead negligible.
- **Collection interval**: Configurable per invocation (e.g., `--interval 1s` shown in README for secondly recording); sub-second resolution is supported (target). Default agent interval is defined in `config/agent.toml` but not reproduced in the snapshot.
- **Async runtime**: Uses Tokio (`tokio = { version = "1.45.1", features = ["full"] }`) for all async I/O, providing a multi-threaded work-stealing executor. Thread-pool sizing inherits Tokio defaults unless overridden in config (not determinable from code).
- **HTTP/2 support**: Axum is compiled with `features = ["http2"]` and the `h2` crate is an explicit dependency, enabling multiplexed, low-latency metric exposition.
- **Compression**: `tower-http` is included with `compression-full` and `decompression-full` features, reducing network overhead on metric scrape endpoints.
- **Parquet columnar storage**: Recorder and Hindsight outputs use Apache Parquet (`parquet = "54.3.1"`, `arrow = "54.3.1"`), providing efficient columnar compression for on-disk metric artifacts.
- **Memory-mapped I/O**: `memmap2` is a dependency, used for efficient file-backed ring buffers (Hindsight mode).
- **Ring buffer (Hindsight)**: Maintains a rolling, high-resolution metrics buffer on disk; size is configurable via `config/hindsight.toml` (exact values not determinable from code).
- **Release profile**: Built with `lto = true` and `codegen-units = 1`, producing maximally optimised single-codegen-unit binaries with full link-time optimisation. Debug symbols are retained (`debug = true`) for profiling without sacrificing runtime performance.
- **CPU affinity**: `core_affinity` crate is present, indicating collection threads may be pinned to specific cores (target — exact pinning policy not determinable from code).
- **Connection pooling**: `reqwest` (blocking feature) is used for the Recorder's HTTP client to the Agent; pool configuration is not visible in the snapshot.

---

## Scalability

- **Deployment model**: Rezolus is a **host-level agent** — one instance per physical or virtual machine. It is not a horizontally scaled service; scale-out is achieved by deploying one instance per host in a fleet.
- **Statelessness of Agent**: The Agent itself is stateless with respect to external services; all metric state is in-process or on local disk. No distributed coordination is required.
- **Hindsight ring buffer**: Scales to the local disk capacity configured; intended for single-host incident capture, not distributed aggregation.
- **Exporter scaling**: The Prometheus-compatible exporter endpoint is served via Axum/HTTP2 and can handle concurrent scrapes from multiple Prometheus instances; no explicit concurrency limits are visible in the snapshot.
- **Replica counts / autoscaling**: _Not determinable from code._ Fleet deployment is managed externally (systemd units are provided; orchestration above that layer is out of scope).
- **Architecture support**: x86_64 and ARM64 explicitly supported (README). Linux kernel ≥ 5.8 required (eBPF CO-RE dependency).
- **Partitioning/sharding**: _Not applicable_ — single-host agent model.

---

## Security

- **Transport security**: _Not determinable from code._ The Agent's HTTP listener and Exporter endpoint configuration (TLS or plaintext) is defined in TOML config files not reproduced in the snapshot. Axum with HTTP/2 is present but TLS termination configuration is not visible.
- **AuthN/AuthZ**: _Not determinable from code._ No authentication middleware (e.g., bearer tokens, mTLS, API keys) is evident in the dependency list. The Exporter is likely intended for scraping within a trusted internal network.
- **Secrets handling**: No secrets management library (e.g., Vault, AWS Secrets Manager SDK) is present in dependencies. Configuration is file-based (TOML in `/etc/rezolus/`), with file permissions set to `644` for config and `755` for the binary. Sensitive values, if any, would reside in plaintext config files.
- **eBPF privilege requirements**: Running eBPF programs requires elevated Linux capabilities (`CAP_BPF`, `CAP_SYS_ADMIN`, or equivalent). The systemd unit files (present in `debian/`) likely specify required capabilities, but their content is not reproduced in the snapshot.
- **Input validation**: The Exporter/Recorder accept HTTP requests (URLs and query parameters); Axum provides basic framework-level input handling. No explicit input validation middleware is visible in the dependency list.
- **Container image**: The production image is built `FROM gcr.io/distroless/cc-debian11`, a minimal, shell-less base image that significantly reduces attack surface. The builder stage uses a private Fastly internal registry.
- **Supply chain**: Dual MIT/Apache-2.0 licensed. Dependencies are pinned by `Cargo.lock` (not shown). Linux-specific native libraries (`libbpf`, `perf-event`) are compiled in only for `cfg(target_os = "linux")`.
- **`parking_lot`**: Used for synchronisation primitives; no security-specific role.

---

## Observability

Rezolus is itself an observability tool; its self-observability posture is as follows:

- **Metrics (self)**: Uses the `metriken` crate (`metriken = "0.7.0"`) as an internal metric registry and `metriken-exposition = "0.12.3"` for exposition. This means Rezolus instruments its own internals using the same metric primitives it exposes for the system.
- **Prometheus endpoint**: The Exporter mode exposes a Prometheus-compatible HTTP endpoint (port configurable in `config/exporter.toml`; exact address/port not determinable from snapshot). Histogram distributions can be converted to summary percentiles to reduce storage overhead.
- **Logging**: Uses `ringlog = "0.8.0"`, a ring-buffer-backed structured logger. Log level and output destination are configurable; exact configuration not determinable from code.
- **Health/readiness endpoints**: _Not determinable from code._ No dedicated `/healthz` or `/readyz` handler is explicitly identifiable from the dependency list or extracted facts; the Axum server may serve such endpoints but this is not confirmed.
- **Tracing**: No distributed tracing library (e.g., `opentelemetry`, `tracing`) is present in the dependency manifest. _Not determinable from code_ whether span-level tracing is implemented.
- **Parquet artifacts**: Recorder and Hindsight modes produce `.parquet` files on local disk, which serve as post-hoc observability artifacts for incident investigation.
- **Viewer**: A local web server (Axum, with `tower-livereload` and `include_dir` for embedded assets) renders Parquet artifacts as an interactive JavaScript dashboard.
- **Backtrace on panic**: `backtrace = "0.3.75"` is a direct dependency, enabling capture and logging of stack traces on unexpected panics.
- **File-system watching**: `notify = "8.0.0"` is present, likely used to watch config files for live reload.

---

## Reliability

- **eBPF program lifecycle**: eBPF programs are loaded and attached at startup; failure to load (e.g., unsupported kernel, missing capabilities) must be handled gracefully. Error handling uses `anyhow` and `thiserror` throughout.
- **Panic handling**: `backtrace` dependency enables structured panic output. The Tokio runtime will catch panics in tasks but a panic in a critical collection thread may halt that sampler; recovery strategy per-sampler is not determinable from code.
- **Ring buffer durability (Hindsight)**: `memmap2`-backed ring buffer persists to disk, providing resilience against process crashes — data already written survives a restart, enabling post-incident capture.
- **Signal handling**: `ctrlc = { version = "3.4.7", features = ["termination"] }` enables clean shutdown on SIGTERM/SIGINT, ensuring in-progress writes (e.g., Parquet files) are flushed before exit.
- **Retry/circuit breaker**: _Not determinable from code._ No explicit retry or circuit-breaker library (e.g., `reqwest-retry`, `failsafe`) is present in the dependency manifest. The blocking `reqwest` client used by the Recorder may rely on OS-level TCP retries only.
- **Idempotency**: Metric collection is read-only with respect to the observed system; re-collection after a restart produces fresh but non-duplicate data. Parquet output files are named per invocation, avoiding overwrite conflicts.
- **Systemd integration**: Three separate systemd unit files are provided (agent, exporter, hindsight), with `enable = true` in the Debian packaging metadata, ensuring automatic restart-on-failure behaviour is available via systemd's `Restart=` directive (exact policy in unit files not reproduced in snapshot).
- **File system change awareness**: `notify` enables config file watching, supporting live reload without requiring a process restart, improving operational resilience.
- **Availability target**: _Not determinable from code._
- **Kernel version dependency**: Requires Linux ≥ 5.8 for eBPF CO-RE support. Running on an older kernel will cause startup failure; no degraded/fallback mode is evident.
