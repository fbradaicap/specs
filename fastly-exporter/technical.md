---
repo: fastly-exporter
spec_type: technical
commit: bebe68cfaf4ceab83b844e46ddd566b2408ba213
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 1ebc9d177a62ad41f42f7a653648848917c9f9eef32d8bda8269bd9a92885943
generated_at: 2026-06-30T14:50:59.294173194+02:00
generator: specsync
---

## Tech Stack

- **Language:** Go (module `github.com/fastly/fastly-exporter`; `go.mod` declares `go 1.25.0`)
- **Runtime:** Compiled to a static binary (`CGO_ENABLED=0`); runs from a `scratch` container image
- **Key frameworks / libraries (direct dependencies):**
  - `github.com/prometheus/client_golang` v1.23.2 — Prometheus instrumentation and metrics exposition
  - `github.com/gorilla/mux` v1.8.1 — HTTP router (metrics and pprof endpoints)
  - `github.com/go-kit/log` v0.2.1 — structured levelled logging
  - `github.com/peterbourgon/ff/v3` v3.4.0 — flag / environment-variable / config-file parsing
  - `github.com/oklog/run` v1.2.0 — actor-model run-group for concurrent goroutine lifecycle management
  - `github.com/json-iterator/go` v1.1.12 — high-performance JSON decoding (used in `pkg/rt`)
  - `github.com/cespare/xxhash` v1.1.0 — fast hashing (used for service-shard partitioning)
  - `golang.org/x/sync` v0.20.0 — `errgroup` for fan-out API polling
- **Notable indirect dependencies:** `github.com/prometheus/common`, `github.com/prometheus/client_model`, `google.golang.org/protobuf` (Prometheus exposition format), `go.yaml.in/yaml/v2` (config file parsing via ff)

---

## Architecture Patterns

- **Architectural style:** Single-binary Prometheus exporter — a **polling agent** that bridges the Fastly REST API to the Prometheus scrape model. There is no inbound data path beyond the Prometheus `/metrics` scrape endpoint.
- **Internal structure:** Loosely **layered**:
  - `cmd/fastly-exporter` — entry point; wires together all components, starts the run-group.
  - `pkg/api` — Fastly API client layer; implements polling caches (`ServiceCache`, `DatacenterCache`, `CertificateCache`, `ProductCache`, `DictionaryInfoCache`). Each cache exposes a `Refresh(ctx)` method and a `Gatherer(namespace, subsystem)` factory that returns a `prometheus.Gatherer`.
  - `pkg/rt` — real-time stats subscriber; consumes the Fastly Real-Time Analytics API (long-poll) and converts JSON payloads to Prometheus metrics.
  - `pkg/prom` — Prometheus metric registration and namespace/subsystem helpers.
  - `pkg/filter` — allow/block list regex filtering for services and metric names.
- **Concurrency model:** `github.com/oklog/run` actors per background refresh ticker; `golang.org/x/sync/errgroup` for parallel per-service API fan-out. All caches are guarded by `sync.Mutex` / `sync.RWMutex`.
- **Pagination:** Implemented via RFC 5988 `Link: rel="next"` header parsing (`pkg/api/link.go`) for services and certificates endpoints.
- **No event bus / message broker** — purely request/response polling.

---

## Database & Data Ownership

This service owns **no persistent datastore**. All state is held in-process as in-memory caches (`[]Service`, `[]Datacenter`, `[]Certificate`, `[]Dictionary`) that are rebuilt on each configurable refresh interval. There are no database migrations, no SQL/NoSQL connections, and no DB tables.

---

## Dependencies

### Runtime — external services / APIs called

| Dependency | Purpose |
|---|---|
| `https://api.fastly.com/service` | List all Fastly services and their active version |
| `https://rt.fastly.com/v1/channel/{id}/ts/{ts}` | Real-Time Analytics API (long-poll per service) |
| `https://api.fastly.com/datacenters` | POP / datacenter metadata |
| `https://api.fastly.com/tls/certificates` | Custom TLS certificate metadata (optional; disabled on 403) |
| `https://api.fastly.com/entitled-products/{product}` | Product entitlement check (origin_inspector, domain_inspector) |
| `https://api.fastly.com/service/{id}/version/{v}/dictionary` | List edge dictionaries per service |
| `https://api.fastly.com/service/{id}/version/{v}/dictionary/{dictId}/info` | Dictionary item count / digest / last-updated |

All outbound calls use a configurable `http.Client` with separate timeouts for `api.fastly.com` (default 15 s) and `rt.fastly.com` (default 45 s). Requests are authenticated via the `Fastly-Key` header.

### Runtime — Go libraries (direct)

See Tech Stack section above. No external caches (Redis, Memcached) or message brokers.

### Build-time only

- `github.com/google/go-cmp` v0.7.0 — test assertion diffing
- `github.com/kylelemons/godebug` v1.1.0 — test utility (indirect)

---

## Deployment Model

### Container image (Dockerfile)

- **Multi-stage build:** `golang:latest` builder → `scratch` final image.
- Binary compiled with `CGO_ENABLED=0` (fully static). Build-time `VERSION` and `BRANCH` are injected via `-ldflags`.
- Final image contains only: the static binary, `/etc/passwd` (for the non-root user), and CA certificates.
- Runs as non-root user `fastly-exporter` (created in the builder stage).
- **Exposed port:** `8080` (TCP); entry point: `/fastly-exporter -listen=0.0.0.0:8080`.
- Published to `ghcr.io/fastly/fastly-exporter`.

### Orchestration

- **Kubernetes / Helm:** Supported via the community chart `prometheus-community/prometheus-fastly-exporter`. No Helm chart source is bundled in this repository.
- **Docker / local:** `docker pull ghcr.io/fastly/fastly-exporter:latest` or build from source with `go build ./cmd/fastly-exporter`.

### Configuration

Flags, environment variables (prefix `FASTLY_EXPORTER_`), or a plain-text config file (`-config-file`). Key settings:

| Flag | Default | Description |
|---|---|---|
| `-token` / `FASTLY_API_TOKEN` | _(required)_ | Fastly API authentication token |
| `-listen` | `127.0.0.1:8080` | Prometheus metrics listen address |
| `-namespace` | `fastly` | Prometheus metric namespace |
| `-service-refresh` | `1m` | Service list polling interval |
| `-datacenter-refresh` | `10m` | Datacenter metadata polling interval |
| `-certificate-refresh` | `6h` | TLS certificate polling interval (0 = disabled) |
| `-product-refresh` | `10m` | Product entitlement polling interval |
| `-dictionary-refresh` | `5m` | Dictionary info polling interval |
| `-api-timeout` | `15s` | HTTP timeout for `api.fastly.com` |
| `-rt-timeout` | `45s` | HTTP timeout for `rt.fastly.com` |
| `-aggregate-only` | `false` | Suppress per-datacenter breakdown |

### Endpoints exposed

| Path | Description |
|---|---|
| `GET /metrics` | Prometheus metrics scrape endpoint |
| `GET /debug/pprof/*` | Go pprof profiling (via `net/http/pprof` blank import) |

### Health / readiness

_Not determinable from code._ No dedicated `/healthz` or `/readyz` handler is evident; liveness can be inferred from the HTTP server accepting connections on port 8080.
