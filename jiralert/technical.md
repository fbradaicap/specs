---
repo: jiralert
spec_type: technical
commit: 3567c32d289b72a6249df61a4f00fbd1018f62e4
model: claude-sonnet-4-6
prompt_version: v1
input_hash: b0230a420a46e50880b88948a4b8d7f32c217c53ec64df989cb4f39d4315723e
generated_at: 2026-06-30T14:54:46.099632837+02:00
generator: specsync
---

## Tech Stack

- **Language:** Go (module declares `go 1.23.0`; Dockerfile builder image uses `golang:1.19`)
- **Runtime:** Statically linked binary running on `quay.io/prometheus/busybox-linux-amd64`
- **Key runtime libraries:**
  - `github.com/andygrunwald/go-jira v1.16.1` — JIRA REST API client
  - `github.com/prometheus/client_golang v1.22.0` — Prometheus metrics exposition
  - `github.com/go-kit/log v0.2.1` — structured logging
  - `gopkg.in/yaml.v3 v3.0.1` — YAML configuration parsing
  - `github.com/trivago/tgo v1.0.7` — utility library
  - `github.com/pkg/errors v0.9.1` — error wrapping
  - `golang.org/x/text v0.25.0` — text processing
  - `github.com/golang-jwt/jwt/v4 v4.5.2` — JWT support (indirect, via go-jira PAT auth)
- **Build tooling:** `make` with `GO111MODULE=on`
- **Test library:** `github.com/stretchr/testify v1.10.0`

## Architecture Patterns

- **Webhook receiver / adapter pattern:** JIRAlert acts as a translation bridge between Prometheus Alertmanager's webhook HTTP API and the Atlassian JIRA API. It receives `POST /alert` requests from Alertmanager and maps them to JIRA issue operations (create, reopen, update, auto-resolve).
- **Request-driven / synchronous HTTP server:** There is no background worker, queue, or event bus. All processing is triggered by inbound HTTP webhook calls and completed synchronously within the request handler.
- **Configuration-driven receiver routing:** Incoming alert payloads carry a `receiver` field that is matched against a named receiver list loaded from a YAML config file. Each receiver encapsulates JIRA connection details and issue-field templates.
- **Go-template rendering:** Issue fields (summary, description, labels, etc.) are rendered at runtime using Go's `text/template` engine populated with Alertmanager notification data, mirroring Alertmanager's own templating model.
- **Internal package layout:**
  - `cmd/jiralert/` — entry point; HTTP handler wiring, flag parsing, logger setup
  - `pkg/alertmanager/` — Alertmanager webhook data structures
  - `pkg/config/` — YAML config loading and receiver lookup
  - `pkg/notify/` — core JIRA notification logic (create/reopen/update issues)
  - `pkg/template/` — Go-template loading and rendering helpers

## Database & Data Ownership

This service owns **no database or persistent datastore**. All state is held externally in the JIRA instance(s) configured per receiver. There are no migrations, ORM models, or embedded stores. JIRA issue state (open/resolved/won't-fix) is read and written exclusively through the JIRA REST API at request time.

## Dependencies

### Runtime dependencies

| Dependency | Purpose |
|---|---|
| Atlassian JIRA instance(s) | External system; issues are created, updated, and transitioned via REST API. Connection configured per receiver (URL, basic auth username/password or Personal Access Token). |
| Prometheus Alertmanager | Upstream caller; sends webhook `POST /alert` payloads. JIRAlert is registered as a webhook receiver in Alertmanager's routing configuration. |
| YAML config file | Local file (`config/jiralert.yml` by default); loaded at startup. |
| Go template file | Path referenced from config; loaded at startup for issue-field rendering. |

### Notable build/indirect dependencies

- `github.com/prometheus/client_golang` — exposes `/metrics` endpoint (Prometheus instrumentation of jiralert itself)
- `github.com/golang-jwt/jwt/v4` — indirect; used by `go-jira` for PAT authentication transport
- `github.com/fatih/structs`, `github.com/google/go-querystring` — indirect; used by `go-jira`
- `net/http/pprof` — imported for debug profiling (activated when `DEBUG` env var is set)
- `github.com/stretchr/testify` — test-only

There are **no message brokers, caches, or other third-party SaaS APIs** involved.

## Deployment Model

### Container image

- **Multi-stage Dockerfile:**
  1. **Builder stage** — `golang:1.19`; copies source, runs `make` with `GO111MODULE=on`, outputs binary to `/tmp/bin` and copies `jiralert` to workspace root.
  2. **Runtime stage** — `quay.io/prometheus/busybox-linux-amd64:latest`; copies only the compiled binary to `/bin/jiralert`.
- **Entrypoint:** `/bin/jiralert`

### Ports

| Port | Protocol | Purpose |
|---|---|---|
| `9097` (default) | HTTP | Main listener — webhook `/alert` endpoint, `/metrics`, `/healthz`, `/config`, `/` |

Overridable via `-listen-address` flag.

### HTTP endpoints

| Path | Method | Description |
|---|---|---|
| `/alert` | POST | Alertmanager webhook receiver; main ingress |
| `/metrics` | GET | Prometheus metrics (via `promhttp`) |
| `/healthz` | GET | Liveness health check; returns `200 OK` |
| `/config` | GET | Renders current loaded configuration |
| `/` | GET | Home/index handler |
| `/debug/pprof/*` | GET | Go pprof profiling (net/http/pprof, active when `DEBUG` env var is set) |

### Configuration & environment

| Flag / Env var | Default | Description |
|---|---|---|
| `-config` | `config/jiralert.yml` | Path to YAML configuration file |
| `-listen-address` | `:9097` | HTTP listen address |
| `-log.level` | `info` | Log level (`debug`, `info`, `warn`, `error`) |
| `-log.format` | `logfmt` | Log format (`logfmt` or `json`) |
| `-hash-jira-label` | `false` | Hash JIRA label content to avoid 255-char limit |
| `-update-summary` | `true` | Allow updating existing issue summary |
| `-update-description` | `true` | Allow updating existing issue description |
| `-reopen-tickets` | `true` | Allow reopening resolved issues |
| `-update-priority` | `true` | Allow updating existing issue priority |
| `-max-description-length` | `32767` | Truncation limit for issue descriptions |
| `DEBUG` (env) | unset | If set, enables block/mutex profiling |

### Orchestration

_Not determinable from code._ No Kubernetes manifests, Helm charts, or Docker Compose files are present in the repository snapshot. An `examples/` directory exists but its contents were not provided.
