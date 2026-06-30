---
repo: pushpin
spec_type: non_functional
commit: e809e7c620218b758d1e71be3dfa3876cb096ac8
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 5399048df56eb38ec42bca7c4b7199dc36a40fb5854dcd7c4301849315faa20c
generated_at: 2026-06-30T14:42:45.985820059+02:00
generator: specsync
---

## Performance

Pushpin is designed as a long-lived connection reverse proxy (HTTP streaming, HTTP long-polling, WebSocket). The following performance-relevant properties are evident:

- **Connection holding / streaming**: The core design explicitly holds connections open for pushing data to clients; credit-based flow control is implemented at the ZMQ message layer (visible in tool handlers, e.g., `resp["credits"] = 200000` and incremental credit grants), constraining per-connection memory usage.
- **I/O model**: The Rust core uses `mio` (non-blocking, event-driven I/O with OS-level polling) and `slab` for O(1) connection slot management, indicating an event-loop architecture rather than a thread-per-connection model.
- **ZMQ transport**: Internal inter-component communication uses ZeroMQ (`zmq = "0.9"`) over IPC sockets, providing low-latency local messaging between components (m2adapter, proxy, handler, connmgr).
- **Compression**: `miniz_oxide` is included as a dependency, indicating support for deflate/gzip compression of response bodies.
- **Benchmarks**: Four criterion benchmarks exist (`memorypool`, `list`, `server`, `client`), confirming performance-critical paths are tracked, but no numeric SLO targets are embedded in the repository.
- **Connection TTL**: Example hold-handler uses a 60,000 ms (60 s) connection TTL (`CONN_TTL = 60000`) and a 60,000 ms expiry sweep interval — illustrative of expected connection lifetime order-of-magnitude.
- **Explicit latency/throughput targets, thread pool sizes, or HTTP timeout configuration**: _Not determinable from code._

## Scalability

- **Horizontal scaling**: The README explicitly documents that multiple mongrel2 instances can connect to the same m2adapter, indicating a fan-in topology designed for horizontal scaling of the front-end tier. The `--merge-output` flag used in the Docker `CMD` implies output merging across multiple sub-processes.
- **Component decomposition**: Pushpin is decomposed into distinct binaries (`pushpin-connmgr`, `m2adapter`, `pushpin-proxy`, `pushpin-handler`, `pushpin-legacy`), allowing individual components to be scaled or replaced independently.
- **Statelessness**: Connection state is held in-process. The architecture is not inherently stateless; long-lived connections are pinned to a specific instance, constraining horizontal scaling of individual proxy nodes to sticky-session or consistent-routing strategies.
- **Replica count / autoscaling configuration**: The `docker/compose.yaml` defines a single replica with no autoscaling configuration. Kubernetes or other orchestration autoscaling directives are absent.
- **Vertical scaling**: No CPU/memory resource limits or requests are declared in any deployment manifest.
- **Partitioning/sharding**: _Not determinable from code._

## Security

- **Transport security (TLS)**: `rustls` (version 0.23, with `ring` backend, TLS 1.2+ enabled via `tls12` feature) and `openssl` (pinned to `=0.10.72`) are both present as dependencies, indicating TLS support for client-facing and/or backend connections. `rustls-native-certs` is included for system certificate trust store integration. TLS certificates for the runner are packaged at `etc/pushpin/runner/certs/`.
- **JWT**: `jsonwebtoken = "9"` is a direct dependency, indicating support for JWT-based authentication or authorization (consistent with GRIP protocol token validation).
- **Input parsing**: `httparse` is used for HTTP request parsing; `ipnet` for IP address/CIDR handling (likely for access control or route matching).
- **Secrets handling**: No secrets manager integration, environment-variable injection of credentials, or vault references are present. Secrets handling beyond file-based certificates is _Not determinable from code._
- **AuthN/AuthZ middleware**: No explicit HTTP middleware chain is visible in the sampled source. JWT validation is present at the library level; how it is applied to specific routes is _Not determinable from code._
- **Container hardening**: The production Docker image runs as a non-root user (`USER 1001`, dedicated `pushpin` user/group with UID/GID 1001), which is a positive security posture.
- **Dependency auditing**: `cargo-audit` is installed in the CI Docker image, indicating automated vulnerability scanning of Rust dependencies.
- **Input validation beyond HTTP parsing**: _Not determinable from code._

## Observability

- **Logging**: The `log = "0.4"` facade is used throughout the Rust codebase. `env_logger` is present as a dev-dependency. Production log configuration (log level, format, output sink) is managed via `pushpin.conf` and the logrotate config (`packaging/debian/pushpin.logrotate`), confirming log rotation is configured for Debian packaging.
- **Metrics**: No metrics library (Prometheus client, StatsD, etc.) is present as a dependency. _Not determinable from code_ whether metrics are emitted.
- **Distributed tracing**: No tracing library (OpenTelemetry, Jaeger client, etc.) is present as a dependency. _Not determinable from code._
- **Health/readiness endpoints**: The extracted endpoints (`GET headers`, `GET type`, `GET more`) do not correspond to conventional health or readiness probe paths. No `/health`, `/ready`, or `/livez` endpoints are identifiable from the manifest. The Docker image does not declare a `HEALTHCHECK` instruction.
- **Debug/profiling**: Four criterion benchmark binaries are present, enabling repeatable micro-benchmark profiling, but no continuous profiling integration is evident.

## Reliability

- **Resilience patterns**:
  - **Keep-alive / heartbeat**: The ZMQ message protocol includes explicit `keep-alive` message types (visible in `keephandler.py`, `holdhandler.py`, `wsechohandler.py`), preventing silent connection drops.
  - **Credit-based flow control**: Back-pressure is enforced via a credit system on ZMQ streams, preventing unbounded buffer growth and acting as a soft rate-limiter.
  - **Handoff protocol**: A `handoff-start` / `handoff-proceed` message exchange (evident in `handoffhandler.py`) allows connection state to be transferred between handler instances, supporting graceful restarts without dropping held connections.
  - **Connection expiry**: Connections are expired after TTL (60 s in example handlers) with periodic sweeps, preventing resource leaks from stale connections.
  - **Panic strategy**: Both `[profile.dev]` and `[profile.release]` set `panic = "abort"`, meaning panics cause immediate process termination rather than stack unwinding; the supervisor (systemd unit or container restart policy) is relied upon for recovery.
- **Configuration hot-reload**: `notify = "7"` (filesystem watching) is a dependency, indicating support for detecting configuration file changes at runtime without a full restart.
- **Retry / circuit-breaker logic**: _Not determinable from code._
- **Idempotency**: Sequence numbers (`seq`) on ZMQ messages provide ordering guarantees and enable duplicate detection, supporting at-least-once delivery semantics.
- **Availability / recovery SLOs**: _Not determinable from code._
