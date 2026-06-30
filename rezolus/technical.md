---
repo: rezolus
spec_type: technical
commit: f8beb6fce7394ff5421f0ad9371adf2bd25903d7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 46fbc1f0c5073dc355a6ecb735431d5c6d3a5672528bcd134a89f460a7891e0e
generated_at: 2026-06-30T14:54:33.524586697+02:00
generator: specsync
---

## Tech Stack

- **Language**: Rust (edition 2021)
- **Runtime version**: Rust 1.93.0 (as specified in the Fastly build Dockerfile base image `focal-rust:1.93.0`)
- **Build system**: Cargo (workspace with sub-crates: `systeminfo`, `xtask`)
- **Notable runtime libraries**:
  - `tokio` 1.45.1 — async runtime (full features)
  - `axum` 0.8.4 — HTTP server framework (HTTP/2 enabled)
  - `libbpf-rs` 0.25.0 / `libbpf-sys` 1.5.1 — eBPF program loading and management (Linux-only)
  - `libbpf-cargo` 0.25.0 — eBPF build-time compilation (Linux-only build dependency)
  - `metriken` 0.7.0 / `metriken-exposition` 0.12.3 — metrics registry and exposition
  - `parquet` 54.3.1 / `arrow` 54.3.1 — columnar data serialisation for recording artifacts
  - `clap` 4.5.40 — CLI argument parsing
  - `perf-event2` 0.7.4 / `perf-event-open-sys2` 5.0.6 — Linux perf event interface (Linux-only)
  - `nvml-wrapper` 0.11.0 — NVIDIA GPU metrics via NVML (Linux-only)
  - `reqwest` 0.12.20 (blocking) — HTTP client for metric scraping
  - `tower` / `tower-http` — HTTP middleware stack (compression, static file serving)
  - `serde` / `toml` — configuration deserialisation
  - `ringlog` 0.8.0 — ring-buffer logger
- **Frontend**: JavaScript (embedded static assets via `include_dir`; served through the Viewer component)
- **Packaging**: `cargo-deb` (Debian `.deb`), `cargo-generate-rpm` (RPM), systemd unit files

## Architecture Patterns

**Architectural style**: Modular CLI-driven agent with multiple operating modes sharing a common binary, structured around an async event loop (Tokio).

**Key patterns**:
- **Plugin/registry pattern** — `linkme` is used for distributed slice registration, allowing metric samplers to self-register at link time without a central registry file.
- **eBPF instrumentation layer** — Linux-specific samplers attach eBPF programs via `libbpf-rs` to collect kernel-space telemetry with minimal overhead; compiled at build time via `libbpf-cargo`.
- **HTTP server + worker** — The Agent exposes an internal metrics HTTP endpoint (via `axum`/`tower`); the Exporter operates as a Prometheus scrape target; both run async workers alongside the HTTP server.
- **Ring-buffer / rolling file** — The Hindsight mode maintains an on-disk rolling buffer (memory-mapped files via `memmap2`) to enable after-the-fact capture.
- **Columnar recording** — The Recorder writes metrics to Apache Parquet files using `arrow`/`parquet` for offline analysis.
- **File-watching** — `notify` is used for configuration hot-reload.

**Operating modes** (all compiled into a single binary, selected via CLI subcommand):
| Mode | Responsibility |
|---|---|
| Agent | Core metric collection daemon |
| Exporter | Prometheus-compatible exposition |
| Recorder | On-demand Parquet file capture |
| Hindsight | Rolling ring-buffer for incident retrospection |
| Viewer | Local web server for Parquet dashboard (JS frontend) |

## Database & Data Ownership

This service owns **no relational or external datastore**. It writes data to:

- **Parquet files** on the local filesystem (Recorder and Hindsight modes) — these are ephemeral artifacts, not a managed database.
- **Memory-mapped ring-buffer files** on disk (Hindsight mode) for the rolling metrics window.

No database migrations, schemas, or external datastore connections are present.

## Dependencies

### Runtime dependencies (external services / infrastructure)
| Dependency | Type | Purpose |
|---|---|---|
| Linux kernel ≥ 5.8 | Host OS | eBPF and perf event APIs required for instrumentation |
| `libelf` / `libbpf` | System library | eBPF object loading at runtime |
| NVIDIA NVML (`libnvidia-ml`) | Optional system library | GPU metrics collection via `nvml-wrapper` |
| Prometheus scraper | External consumer | Scrapes the Exporter's HTTP endpoint |
| Rezolus Agent HTTP endpoint (`http://localhost:4241` default) | Internal | Recorder and Hindsight connect to the Agent to pull metrics |

### Build-time dependencies
| Dependency | Purpose |
|---|---|
| `clang` | Required to compile eBPF C programs via `libbpf-cargo` |
| `libelf-dev` | Required by `libbpf-sys` at build time |
| `cargo-deb` | Produces `.deb` package |
| `libbpf-cargo` | Compiles and links eBPF skeletons during `build.rs` |

### Notable library dependencies (runtime)
- `reqwest` (blocking) — HTTP client used by Recorder/Hindsight to pull metrics from the Agent
- `axum` + `tower-http` — HTTP server for Agent and Exporter endpoints
- `parquet` + `arrow` — Columnar format for artifact files
- `metriken` + `metriken-exposition` — Internal metrics storage and Prometheus text/protobuf serialisation
- `rmp-serde` — MessagePack serialisation (internal wire format)

No external message brokers, caches, or third-party SaaS APIs are used.

## Deployment Model

### Container image
- **Build**: Multi-stage Docker build defined in `fastly-build/Dockerfile`
  1. **Builder stage**: `container-registry.secretcdn.net/fastly/focal-rust:1.93.0` (Ubuntu Focal + Rust 1.93.0); installs `clang` and `libelf-dev`; runs `cargo deb` to produce a `.deb` package and the release binary.
  2. **Runtime stage**: `gcr.io/distroless/cc-debian11` — minimal distroless image; copies only `/rezolus` binary.
- **Entrypoint**: `/rezolus`
- **Build args**: `DESTDIR` (deb output path), `PKG_VERSION` (package version string)

### Linux packages
- **Debian/Ubuntu**: `.deb` via `cargo-deb`; installs binary to `/usr/bin/rezolus`, configs to `/etc/rezolus/`, and three systemd unit files.
- **RHEL/RPM**: `.rpm` via `cargo-generate-rpm`; same layout.

### Systemd units
| Unit file | Purpose |
|---|---|
| `rezolus.service` | Agent daemon |
| `rezolus-exporter.service` | Prometheus exporter |
| `rezolus-hindsight.service` | Hindsight ring-buffer daemon |

### Configuration
- TOML configuration files at `/etc/rezolus/agent.toml`, `exporter.toml`, `hindsight.toml`
- Runtime environment configuration is file-based (TOML); hot-reload via `notify`

### Ports
- Agent default listen address: `http://localhost:4241` (referenced in README `record` example); exact port is configured via TOML.
- Exporter exposes a Prometheus-compatible HTTP endpoint; specific port is configured via `exporter.toml`.

### Supported platforms
- Architectures: `x86_64`, `aarch64` (ARM64)
- OS: Linux kernel ≥ 5.8 (bare-metal and cloud)

### Health / readiness endpoints
_Not determinable from code._
