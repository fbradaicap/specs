---
repo: fastly-log-analytics
spec_type: technical
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:57:16.978197709+02:00
generator: specsync
---

## Tech Stack

**Backend**
- **Language/Runtime:** Python 3.13 (CPython, `python:3.13-slim-bookworm`)
- **Package manager:** `uv` 0.6.14 with `uv.lock` (frozen installs)
- **Web framework:** FastAPI (with Uvicorn as ASGI server, inferred from project structure and `uvicorn` references in source)
- **Analytics engine:** DuckDB (in-process, `DUCKDB_MEMORY_LIMIT=4GB`; `httpfs` extension for object-store access)
- **Table format:** Apache Iceberg (PyIceberg, inferred from `backend/core/iceberg` module)
- **Scheduler:** APScheduler (inferred from scheduler references in source)
- **SSE streaming:** `sse-starlette`
- **Memory allocator:** jemalloc 2 (preloaded via `LD_PRELOAD`)
- **VCL linter:** Falco 2.3.0 (binary, installed in image)

**Edge Compute (Wasm)**
- **Language/Runtime:** Rust (edition 2021), compiled to `wasm32-wasip1`
- **Fastly SDK:** `fastly = "0.11"`
- **Notable crates:** `aes-gcm 0.10`, `base64 0.22`, `hex 0.4`, `getrandom 0.2`
- **Build profile:** size-optimised (`opt-level = "s"`, LTO, `strip = "symbols"`, `panic = "abort"`)

**Frontend**
- **Language/Runtime:** TypeScript 5.x, Node.js ≥ 24
- **Framework:** Next.js 16.2.9 (React 19, App Router, standalone output)
- **State management:** Zustand 5, TanStack Query 5
- **UI components:** shadcn/ui, Radix UI primitives, Base UI, Lucide React, cmdk
- **Data visualisation:** Plotly.js (cartesian-dist-min), react-plotly.js, MapLibre GL
- **Tables/Virtualisation:** TanStack Table 8, TanStack Virtual 3
- **SQL editor:** CodeMirror 6 (`@codemirror/lang-sql`), `@uiw/react-codemirror`
- **Date handling:** date-fns 4, date-fns-tz 3
- **Drag-and-drop:** dnd-kit (core, sortable, modifiers, utilities)
- **API client:** `openapi-fetch`; types auto-generated via `openapi-typescript`
- **URL state:** nuqs 2
- **Styling:** Tailwind CSS 4, tw-animate-css, tailwind-merge, clsx, class-variance-authority
- **Geo:** MapLibre GL 5, topojson-client/server
- **RUM:** web-vitals 5

**Testing / tooling**
- Vitest 4 + `@vitest/coverage-v8`, Testing Library (React, DOM, user-event), jsdom, vitest-axe
- Playwright 1.61 (E2E + accessibility via `@axe-core/playwright`), msw 2 (API mocking)
- ESLint 9 + `eslint-config-next`, knip 6 (dead-code detection)
- `osv-scanner` (dependency vulnerability gate)
- `uv run python` scripts for CI gates and operational tasks

**Reverse proxy / ingress**
- Caddy 2 (Alpine) with `caddy-ratelimit` plugin (custom build via `xcaddy`)

---

## Architecture Patterns

**Overall style:** Layered monorepo with three distinct runtime tiers:

1. **Backend (FastAPI + DuckDB)** — REST API server following a layered/modular structure: `routers` → `core` → `repositories` → `utils`. Routers are split by domain into sub-packages (admin, downloads, iceberg, ingest, etc.) and registered via side-effect imports on a shared `APIRouter`. Dependency injection via FastAPI `Depends` (e.g. `get_source`).

2. **Frontend (Next.js)** — Server-side rendering + client-side React SPA. API types are generated from the backend's FastAPI OpenAPI schema at build time (`generate_openapi.py` → `openapi-typescript`), enforcing a contract-first API boundary. Client state split between TanStack Query (server state) and Zustand (local UI state).

3. **Edge Compute (Fastly Wasm)** — Stateless, instance-per-request session-anomaly scorer running on Fastly Compute@Edge. Loads a scoring matrix from a KV Store in a hand-rolled binary format (FSM1) to minimise cold-start size. Generates session IDs, encrypts cookies (AES-GCM), and emits `X-Edge-Sid` response headers.

**Key patterns:**
- **Admin/worker separation:** Long-running operations (ingest, metadata sync, Iceberg commit, local view rebuild) are dispatched to daemon threads and return a `run_id` for async polling (202 Accepted pattern). A `start_or_resume_cron` helper coalesces duplicate triggers.
- **Scheduled background jobs:** APScheduler drives recurring crons (ingest, compaction, commit, metric snapshots, gap-heal). A bounded graceful shutdown with 60 s drain budget is paired with a 75 s Docker `stop_grace_period`.
- **Stale-while-revalidate caching:** Directory size scans and Fastly Stats API responses use in-process TTL caches with coalescing cold locks to avoid thundering-herd on expensive operations.
- **Streaming responses:** ZIP downloads and SSE progress streams use a producer thread writing to an `_AbortableQueue`, consumed by a FastAPI `StreamingResponse`.
- **Single shared network namespace:** Caddy, frontend (Next.js), and backend all share the backend container's network namespace in both dev (compose) and prod (`network_mode: host`), so the backend's loopback-trust model (`is_request_remote`) works consistently.
- **OpenAPI contract enforcement:** A `generate_openapi.py` script is a load-bearing build step; both the frontend Dockerfile and `gen:types` npm script run it before compilation.

---

## Database & Data Ownership

**Datastores owned by this service:**

| Store | Type | Purpose |
|---|---|---|
| Per-service DuckDB files (`.duckdb`) | Embedded analytical database | Local query layer over Parquet cache; exclusive write lock per service |
| Local Parquet cache (`/app/cache`) | Columnar files on local disk | Raw log data + pre-computed rollups (hourly top-N, origin summary, network RTT/speed, perf latency, verified bots) organised by service/partition |
| Apache Iceberg tables | Open table format on FOS (object storage) | Long-term durable log storage; local DuckDB view re-created from Iceberg catalog on sync |
| Per-service SQLite (`/app/data`) | Embedded relational database | Cron run metadata, metric snapshots, usage log, ingested-file accounting |
| Scoring matrix (`compute/scorer/matrix.json` / `matrix.default.json`) | Binary file (FSM1 format) | Edge session-anomaly scoring weights; loaded into Wasm KV Store |
| Service configs (`/app/configs`) | YAML/JSON files | Per-service configuration including FOS credentials, CDN URLs, retention settings |

No DB migrations directory was detected in the snapshot; schema management is _Not determinable from code._ for DuckDB/SQLite specifics.

**Ownership boundaries:** This service is the sole writer to all local Parquet, DuckDB, and SQLite stores. Iceberg table writes go via PyIceberg to FOS (object storage). The Wasm edge component reads-only from the Fastly KV Store.

---

## Dependencies

**Runtime — external services / APIs:**
- **Fastly Stats API** — queried for log-accounting reconciliation (`/stats` endpoint)
- **Fastly Compute API** — used to publish the Wasm scorer package to customer Compute services (`enable_scoring`)
- **FOS (Fastly Object Storage / S3-compatible)** — primary durable log store; accessed via `boto3`/`botocore`-compatible S3 client (`_get_fos_client`)
- **CDN (customer-configured)** — optional CDN URL for file downloads; authenticated via `x-fastly-key` header
- **Fastly KV Store** — read by the Wasm edge scorer at request time

**Runtime — infrastructure:**
- **Caddy 2 + caddy-ratelimit** — TLS termination and rate limiting (reverse proxy, same host)
- **jemalloc 2** — memory allocator preloaded for the Python backend process
- **Falco 2.3.0** — VCL static analyser binary, required when `SCORING_REQUIRE_FALCO=1`

**Runtime — notable Python libraries:**
- FastAPI, Uvicorn (web framework + ASGI server)
- DuckDB (in-process analytics; `httpfs` extension for FOS access)
- PyIceberg (Iceberg table management)
- APScheduler (background job scheduling)
- sse-starlette (Server-Sent Events)

**Build-time:**
- `uv` 0.6.14 — Python dependency resolution and virtual environment management
- Node.js 24 + npm — frontend build
- `openapi-typescript` — generates TypeScript API types from FastAPI OpenAPI schema
- `xcaddy` — custom Caddy binary build with ratelimit plugin
- Rust toolchain + `cargo` — Wasm scorer build (not present in production image)
- `osv-scanner` — dependency CVE scanning (CI gate)

**Frontend runtime dependencies** (notable): TanStack Query/Table/Virtual, Next.js 16, React 19, Zustand, Plotly.js, MapLibre GL, CodeMirror 6, openapi-fetch, nuqs, dnd-kit, shadcn/ui, Radix UI, date-fns, web-vitals.

**Development/test dependencies**: Vitest 4, Playwright 1.61, msw 2, Testing Library, axe-core, knip, ESLint 9.

---

## Deployment Model

**Container images:**

| Image | Dockerfile | Base | Port |
|---|---|---|---|
| Backend | `backend/Dockerfile` | `python:3.13-slim-bookworm` (multi-stage) | 8000 (internal only) |
| Frontend | `frontend/Dockerfile` | `node:24-slim` (multi-stage, 3 stages) | 3000 (internal only) |
| Caddy | `caddy/Dockerfile` | `caddy:2-builder` → `caddy:2-alpine` | 80 (externally published) |

**Build stages:**
- **Backend:** builder stage installs Python deps via `uv sync --frozen --no-dev`; runner stage copies `.venv`, source, Falco binary, and default scoring matrix. Non-root user `app` (UID/GID 1000).
- **Frontend:** `api-schema` stage runs `generate_openapi.py` against the backend source to emit `openapi.json`; `builder` stage runs `npm ci` + `openapi-typescript` + `next build`; `runner` stage uses Next.js standalone output. Non-root user `nextjs` (UID/GID 1001).
- **Caddy:** `xcaddy build --with github.com/mholt/caddy-ratelimit`; custom non-root user `caddy` with `CAP_NET_BIND_SERVICE` filecap.

**Orchestration:**
- `docker-compose.yml` — local dev; backend publishes `:80`, frontend and Caddy join backend's network namespace (`network_mode: "service:backend"`). Single bridge network `app-network`.
- `docker-compose.prod.yml` — production override (referenced in source but not provided); uses `network_mode: host`.
- No Kubernetes/Helm manifests are present in the snapshot.

**Volume mounts (backend):**
- `./configs:/app/configs` — service configuration files
- `./data:/app/data` — SQLite databases and durable data
- `./cache:/app/cache` — local Parquet/DuckDB cache

**Environment configuration (backend):**
- `DUCKDB_MEMORY_LIMIT` (default `4GB`)
- `ARROW_DEFAULT_MEMORY_POOL=system`
- `LOCAL_HOSTS` — additional hostnames for the loopback trust allowlist
- `SCORING_REQUIRE_FALCO` — hard-fail VCL validation if Falco binary absent
- `OTEL_EXPORTER` — telemetry exporter mode (must not be `console` in deployed env, enforced by CI gate ADR-08)

**Health / readiness:**
- Backend healthcheck: `GET http://localhost:8000/api/health` (interval 30s, timeout 5s, 3 retries, start period 15s)
- Admin health snapshot endpoint: `GET /admin/health-snapshot` (returns load, memory, disk, DuckDB pool stats, scheduler liveness, cron in-flight runs)
- Frontend depends on `backend` `service_healthy` condition before starting.

**Edge (Wasm scorer):**
- Built separately via `scripts/scoring/deploy_wasm.sh` (`make scorer-package`); published to Fastly Compute via the Fastly API. Not deployed as a container.
