---
repo: pushpin
spec_type: technical
commit: e809e7c620218b758d1e71be3dfa3876cb096ac8
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 5399048df56eb38ec42bca7c4b7199dc36a40fb5854dcd7c4301849315faa20c
generated_at: 2026-06-30T14:42:45.985820059+02:00
generator: specsync
---

## Tech Stack

- **Language:** Rust (primary implementation), with Python and JavaScript used for tooling and example handler scripts
- **Rust edition:** 2018; **minimum supported Rust version:** 1.75
- **Package version:** 1.42.0-dev
- **Key Rust runtime libraries:**
  - `zmq` 0.9 — ZeroMQ messaging (core IPC/messaging transport)
  - `mio` 1 — non-blocking I/O event loop
  - `rustls` 0.23 — TLS (with `ring` backend, TLS 1.2 support)
  - `openssl` 0.10.72 — OpenSSL bindings
  - `httparse` 1.7 — HTTP/1.x parsing
  - `clap` 4.3.24 — CLI argument parsing
  - `config` 0.14 — configuration file handling
  - `serde` / `serde_json` 1.0 — serialization
  - `jsonwebtoken` 9 — JWT support
  - `signal-hook` 0.3 — Unix signal handling
  - `notify` 7 — filesystem change notifications
  - `url` 2.3 — URL parsing
  - `log` 0.4 — logging facade
  - `time` 0.3.41 — time/date formatting
  - `sha1` 0.10, `base64` 0.13, `miniz_oxide` 0.6 — crypto/compression utilities
  - `rustls-native-certs` 0.6 — native certificate store integration
  - `socket2` 0.4, `libc` 0.2, `ipnet` 2 — low-level networking
  - `slab` 0.4, `arrayvec` 0.7 — memory management
- **Build tooling:** Cargo (with `cbindgen` 0.27 for C header generation, `pkg-config` 0.3), `make`, `qmake`/Qt6, g++ (C++ component build)
- **C++/Qt dependency:** Qt 6 (`qt6-base-dev`) and Boost are required at build time, indicating a mixed Rust + C++ codebase
- **Python tooling:** `zmq` (pyzmq), `tnetstring` — used exclusively in development/testing handler scripts

## Architecture Patterns

Pushpin is a **reverse proxy** designed for realtime web services, implementing the **GRIP (Generic Realtime Intermediary Protocol)** pattern. The overall architecture is:

- **Multi-process / component pipeline:** The Cargo manifest defines several distinct binaries that form a processing pipeline:
  - `pushpin` — main orchestrator / runner
  - `pushpin-proxy` — HTTP/WebSocket proxy layer (client-facing)
  - `pushpin-handler` — GRIP/subscription hold logic
  - `pushpin-connmgr` — connection manager
  - `m2adapter` — Mongrel2 protocol adapter (converts Mongrel2 protocol to ZMQ-HTTP / ZHTTP)
  - `pushpin-legacy` — legacy compatibility shim
  - `pushpin-publish` — publish-side CLI tool
- **Event-driven / message-passing:** Internal components communicate via **ZeroMQ** sockets (PULL, PUB, ROUTER, DEALER, REP patterns), using **TNetString**-encoded messages. This is consistent throughout tooling and source samples.
- **ZHTTP protocol:** An internal ZeroMQ-based HTTP/WebSocket framing protocol is used for communication between the proxy and backend handlers.
- **Non-blocking I/O:** `mio` is used for the Rust event loop, supporting high connection concurrency.
- **Streaming / long-lived connection support:** The credit-based flow-control model visible in the handler tools reflects support for HTTP streaming and WebSocket hold connections.
- **Mixed Rust + C++ implementation:** `do-qmake` and `do-cpp-build` Cargo features, along with Qt6 and Boost build dependencies, indicate that some internal components (likely legacy proxy logic) are implemented in C++ and linked into the Rust build via `cbindgen`-generated bindings.
- **Configuration-driven routing:** Routes and configuration files (`pushpin.conf`, `routes`) drive backend proxying behaviour.

## Database & Data Ownership

This service owns **no database or persistent datastore**. No migrations, ORM models, or database client libraries are present in the manifests. Pushpin is a stateful-in-memory reverse proxy; connection state is held in process memory and communicated via ZeroMQ.

## Dependencies

### Runtime Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| ZeroMQ (`libzmq3-dev` / `zmq` crate) | Runtime | Core inter-process/inter-component messaging |
| OpenSSL (`libssl-dev` / `openssl` crate) | Runtime | TLS termination |
| `rustls` | Runtime | Alternative TLS stack (ring backend) |
| Qt 6 (`libqt6core6`, `libqt6network6`) | Runtime | C++ component runtime (networking/core Qt) |
| `libzmq5` | Runtime container | ZMQ shared library in Docker image |
| OS signal handling (`signal-hook`) | Runtime | Graceful shutdown/reload on POSIX signals |
| Filesystem notifications (`notify`) | Runtime | Config/route file hot-reload |

### Build-time Dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `qt6-base-dev`, `g++`, Boost | Build | C++ component compilation |
| `cbindgen` | Build | Generate C headers from Rust for C++ interop |
| `pkg-config` | Build | Library discovery |
| `cargo-audit` | CI/Dev | Security audit of Cargo dependencies |
| `clang-format-20` | CI/Dev | C++ code formatting |
| `black` | CI/Dev | Python code formatting |
| `rustfmt`, `clippy` | CI/Dev | Rust formatting and linting |

### Development / Test Dependencies

| Dependency | Purpose |
|---|---|
| `criterion` 0.5 | Benchmarking (memorypool, list, server, client benches) |
| `env_logger`, `test-log` | Test logging |
| `tempfile` | Temporary file handling in tests |
| Python `zmq` + `tnetstring` | Handler simulation scripts for integration testing |

No calls to external third-party SaaS APIs or other microservices are detected in the manifests.

## Deployment Model

### Container Image

- **Build:** Multi-stage Docker build (`docker/Dockerfile`). The `build` stage uses `ubuntu:24.10`, installs build dependencies (`bzip2`, `pkg-config`, `make`, `g++`, `rustc`, `cargo`, `libssl-dev`, `qt6-base-dev`, `libzmq3-dev`, `libboost-dev`), downloads the release tarball from GitHub Releases (`https://github.com/fastly/pushpin/releases/download/v${VERSION}/...`), and builds via `make RELEASE=1 PREFIX=/usr CONFIGDIR=/etc`. The runtime stage is a minimal `ubuntu:24.10` image with only `libqt6core6`, `libqt6network6`, and `libzmq5`.
- **Entrypoint:** `docker-entrypoint.sh`; default command: `pushpin --merge-output`
- **User:** Runs as non-root user `pushpin` (UID/GID 1001)
- **CI Dockerfile** (`.github/Dockerfile`): Based on `ubuntu:24.04`, installs `clang-format-20`, build dependencies, and a pinned Rust stable toolchain (minimum `1.75.0`). Used for CI lint, format, audit, and build checks.

### Exposed Ports

| Port | Protocol | Purpose |
|---|---|---|
| 7999 | HTTP | Inbound client HTTP connections (forwarded to app) |
| 5560 | ZMQ PULL | Receive published messages |
| 5561 | HTTP | Receive messages and commands via HTTP |
| 5562 | ZMQ SUB | Receive published messages (subscribe) |
| 5563 | ZMQ REP | Receive commands |

### Orchestration

- **Docker Compose** (`docker/compose.yaml`): Single-service compose definition referencing image `fanout/pushpin:1.41.0-1`. Suitable for local development.
- **Kubernetes:** _Not determinable from code._ (no Helm charts or k8s manifests present in the snapshot)

### Configuration

- Primary config: `/etc/pushpin/pushpin.conf` (installed from `dist/etc/pushpin/pushpin.conf`)
- Routing rules: `/etc/pushpin/routes`
- Internal config: `/usr/lib/pushpin/internal.conf`
- TLS certs: `/etc/pushpin/runner/certs/`
- Config file hot-reload is supported via filesystem notifications (`notify` crate)
- Log rotation: managed via `/etc/logrotate.d/pushpin`
- Init: systemd unit (auto-generated by `cargo-deb`) and SysV init script (`/etc/init.d/pushpin`)

### Health / Readiness Endpoints

_Not determinable from code._
