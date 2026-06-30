---
repo: rezolus
spec_type: behavioral
commit: f8beb6fce7394ff5421f0ad9371adf2bd25903d7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 46fbc1f0c5073dc355a6ecb735431d5c6d3a5672528bcd134a89f460a7891e0e
generated_at: 2026-06-30T14:54:33.524586697+02:00
generator: specsync
---

## API Contracts

Rezolus is a systems telemetry agent, not a network microservice. It does not expose a general-purpose API server in the traditional sense. However, based on the README and dependency manifest (specifically `axum`, `tower`, `tower-http`, and `metriken-exposition`), the following synchronous HTTP interfaces are evidenced per operating mode:

**Protocol: HTTP/1.1 and HTTP/2** (axum with `http2` feature; `h2` crate present)

### Agent

The Agent exposes a metrics endpoint used by the Recorder and other consumers. Based on the README usage example:

```bash
rezolus record --interval 1s --duration 15m http://localhost:4241 rezolus.parquet
```

The Agent listens on **port 4241** (default).

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` | (metrics endpoint) | Expose collected performance metrics for polling by Recorder/Exporter | None | Serialized metrics payload (format per mode; see Input/Output Formats) |

The exact paths are _Not determinable from code_ given the snapshot provided (no route definitions in the extracted facts or shown source files).

### Exporter

The Exporter serves a **Prometheus-compatible** scrape endpoint. Based on `metriken-exposition` crate usage:

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` | `/metrics` | Prometheus scrape endpoint exposing system telemetry metrics | None | Prometheus text exposition format |

The exact path is _Not determinable from code_ with certainty; `/metrics` is the conventional default for `metriken-exposition`.

### Viewer

The Viewer operates as a local HTTP server (`axum`, `tower-http` with `fs` feature, `tower-livereload`) serving a web-based dashboard for Parquet artifact inspection. Specific routes are _Not determinable from code._

---

## Event Schemas

_Not determinable from code._

No message broker, event bus, Kafka topics, RabbitMQ queues, or any asynchronous publish/subscribe mechanisms are evidenced in the dependency manifest, extracted facts, or README. Rezolus operates via synchronous HTTP polling (Agent → Recorder/Exporter pull model) rather than event-driven messaging.

---

## Input / Output Formats

### Agent Output (metrics polling)

- **Serialization**: MessagePack (`rmp-serde` crate) is evidenced as the primary wire format for metric snapshots consumed by the Recorder.
- The `metriken` and `metriken-exposition` crates manage internal metric registration and exposition.
- Histogram distribution data is supported natively (`histogram` crate).

### Exporter Output (Prometheus)

- **Content-Type**: `text/plain; version=0.0.4` (Prometheus text exposition format), as is conventional for `metriken-exposition`.
- Histogram distributions can be converted to summary percentiles (explicitly stated in README).

### Recorder Output

- **Format**: Apache Parquet (`.parquet` files) via the `parquet` and `arrow` crates.
- Column-oriented storage with Arrow in-memory representation.
- Files are written locally to disk as specified by the CLI invocation.

### Viewer Input

- Reads `.parquet` files produced by Recorder or Hindsight modes.
- Serves a web dashboard; static assets are embedded via `include_dir`.

### Configuration

- **Format**: TOML (`.toml` config files; `toml` crate).
- Config files: `config/agent.toml`, `config/exporter.toml`, `config/hindsight.toml`.

### Serialization Summary

| Interface | Format |
|-----------|--------|
| Agent metrics wire format | MessagePack (`rmp-serde`) |
| Exporter scrape endpoint | Prometheus text |
| Recorder artifact | Apache Parquet |
| Configuration files | TOML |
| Internal/debug serialization | JSON (`serde_json`) |

---

## Error Handling

Specific error payload structures, HTTP status code mappings, and validation behaviour are _Not determinable from code_ from the snapshot provided (no source file contents with route/handler implementations are included).

The following is evidenced from the dependency manifest:

- **`anyhow`** (v1.0.98): Used for ergonomic error propagation throughout the application; implies errors are wrapped with contextual messages rather than typed error hierarchies at the application boundary.
- **`thiserror`** (v2.0.12): Used to define typed, structured error types in specific subsystems.
- **`backtrace`** (v0.3.75): Backtrace capture is enabled, suggesting crash/panic diagnostics are surfaced with stack traces.
- **`axum`** error handling conventions apply to HTTP responses where axum is used; specific handler-level error mappings are _Not determinable from code._

---

## Versioning

- **Application version**: `5.2.2` as declared in `Cargo.toml`.
- **API versioning strategy**: _Not determinable from code._ No URI version prefix (e.g., `/v1/`), version negotiation header, or schema evolution mechanism is evidenced in the available snapshot.
- The Parquet artifact format versioning follows the Apache Parquet and Arrow specification versions (`arrow = "54.3.1"`, `parquet = "54.3.1"`); schema evolution for stored artifacts is _Not determinable from code._
