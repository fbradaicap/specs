---
repo: fastly-log-analytics
spec_type: technical
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:58:31.107203435+02:00
generator: specsync
---

## Tech Stack

| Layer | Technology | Version / Notes |
|---|---|---|
| Backend language | Python | 3.13 (slim-bookworm base image) |
| Backend framework | FastAPI | Via `pyproject.toml`; `uv` for dependency/env management (v0.6.14) |
| Backend ASGI server | Uvicorn | Inferred from FastAPI deployment pattern |
| Backend data engine | DuckDB | In-process analytical engine; `DUCKDB_MEMORY_LIMIT=4GB`; `httpfs` extension for object-storage reads |
| Backend memory allocator | jemalloc 5.3 | `LD_PRELOAD=libjemalloc.so.2` at container start |
| Edge compute language | Rust | Edition 2021, compiled to `wasm32-wasip1` for Fastly Compute |
| Edge compute runtime | Fastly Compute (Wasm) | `fastly = "0.11"` SDK; session-anomaly scorer |
| Edge crypto | `aes-gcm = "0.10"`, `base64 = "0.22"`, `hex = "0.4"`, `getrandom = "0.2"` | Pure-Rust, Wasm-compatible |
| Frontend language | TypeScript / JavaScript | TypeScript `^5.9`, Node.js `>=24` |
| Frontend framework | Next.js | `16.2.9` (App Router, standalone output mode) |
| Frontend state management | Zustand `^5.0.12`, TanStack Query `^5.101.1` | Server state + client store |
| Frontend UI components | shadcn `^4.8.1`, Radix UI, Base UI, Lucide React | Tailwind CSS v4 design system |
| Frontend data tables | TanStack Table `^8.21.3`, TanStack Virtual `^3.14.2` | Virtualised large datasets |
| Frontend charting | Plotly.js (cartesian dist min) `^3.6.0`, react-plotly.js `^4.0.0` | Time-series and scatter charts |
| Frontend mapping | MapLibre GL `^5.24.0`, topojson-client `^3.1.0` | POP location visualisation |
| Frontend SQL editor | CodeMirror + `@codemirror/lang-sql` | In-app query editor |
| Frontend API client | `openapi-fetch ^0.17.0` | Generated from FastAPI OpenAPI schema |
| Frontend drag-and-drop | `@dnd-kit/core`, sortable, modifiers | Dashboard panel reordering |
| Frontend URL state | `nuqs ^2.8.9` | Query-string driven UI state |
| Frontend date handling | `date-fns ^4.2.1`, `date-fns-tz ^3.2.0` | |
| Frontend web vitals | `web-vitals ^5.3.0` | RUM collection |
| VCL linter | Falco `2.3.0` | Fastly VCL static analyser; pinned binary in backend image |
| Reverse proxy / ingress | Caddy 2 + `caddy-ratelimit` plugin | Custom-built image |
| Test – frontend unit | Vitest `^4.0.0`, Testing Library, jsdom, msw | |
| Test – frontend e2e | Playwright `^1.61.1`, `@axe-core/playwright` | Chromium + accessibility |
| Test – vulnerability | `osv-scanner` (via `check_osv.py`) | CI gate |
| Build tooling | `uv` (Python), `npm ci` (Node), `xcaddy` (Caddy), `cargo` (Rust) | |

---

## Architecture Patterns

**Overall style:** Multi-tier, containerised monorepo with a distinct edge-compute component.

**Tiers:**

1. **Edge / Fastly Compute (Rust/Wasm):** A per-request, instance-per-request Wasm binary (`session-scorer`) deployed to the Fastly CDN edge. It performs session anomaly scoring using an AES-GCM encrypted cookie for session continuity, a KV-Store-loaded scoring matrix, and emits a `X-Edge-Sid` header. Cold-start optimised (size-optimised release profile, no serde).

2. **Backend (Python/FastAPI):** Layered REST API server.
   - **Router layer** (`backend/routers/`): FastAPI `APIRouter` instances, split into sub-packages (`admin/`, etc.) with a shared router instance to avoid circular imports. Endpoints are registered by side-effect on import.
   - **Core layer** (`backend/core/`): DuckDB connection pool, Iceberg table management, local compaction, rollup computation, Fastly API client.
   - **Scheduler layer** (`backend/scheduler.py`): APScheduler-backed cron system for periodic ingest, commit, metadata sync, compaction, and gap-heal jobs. Bounded 60 s drain on SIGTERM.
   - **Repository layer** (`backend/repositories/`): Dashboard query cache and invalidation.
   - **Dependency injection** (`backend/deps.py`): FastAPI `Depends`-based source (service config) resolution per request.
   - **SSE streaming:** `sse-starlette` used for long-running admin operations.
   - **Stale-while-revalidate caching:** Thread-safe TTL caches with background refresh for directory sizes, Fastly Stats API responses, and DuckDB counts (coalesced cold-locks to prevent thundering herds).

3. **Frontend (Next.js/React):** Server-side rendered and statically generated pages. TypeScript types are auto-generated from the FastAPI OpenAPI schema at build time (`openapi-typescript`). TanStack Query manages server state; Zustand manages client-side UI state. URL state is serialised via `nuqs`.

4. **Ingress (Caddy):** Single externally-facing socket; rate-limiting via `caddy-ratelimit` plugin. In production (`network_mode: host`); in development, all three containers share the backend's network namespace so loopback trust classification works correctly.

**Key patterns:**
- **OpenAPI-first type safety:** Schema generated from FastAPI routes; frontend types derived from it at build time.
- **CQRS-lite:** Read endpoints return cached/pre-rolled-up data; write/admin endpoints trigger async cron workers returning a `run_id` for polling (202 Accepted pattern).
- **Stale-while-revalidate:** Multiple layers (server-side TTL memo caches, React Query `staleTime`, CDN).
- **Path-safety / cross-tenant guards:** Explicit prefix checks and `path_within_dir` validation on all file-serving endpoints.
- **Admin trust by network locality:** Backend classifies requests from loopback as local admin (`remote_access.py:is_request_remote`); enforced via shared network namespace in Compose.

---

## Database & Data Ownership

**Datastores owned by this service:**

| Store | Type | Purpose |
|---|---|---|
| Per-service DuckDB (`.duckdb` file) | Embedded analytical DB (file-per-service, exclusive lock) | Local analytical view over ingested Parquet; connection pool managed in `backend/core/duckdb.py` |
| Local Parquet cache (`/app/cache`) | Column-store files on disk | Raw and rolled-up log data partitioned by service/field/hour/day; compacted by local compaction cron |
| Iceberg table (remote object storage) | Apache Iceberg on FOS (S3-compatible) | Durable log archive; committed from local buffer by scheduler |
| Per-service SQLite (`cron_runs`, `metric_snapshots`, etc.) | Embedded relational DB | Cron run history, scheduler liveness metrics, usage logging |
| Config files (`/app/configs`) | YAML/JSON on disk | Per-service source configurations (Fastly service IDs, bucket, CDN URL, credentials) |
| Scoring matrix (`compute/scorer/matrix.json` / `matrix.default.json`) | Binary FSM1 / JSON on disk | Edge session-scoring weights; loaded into Wasm KV Store |

**No DB migrations detected** (no SQL migration files found in the snapshot). Schema is managed implicitly via DuckDB DDL in application code and Parquet schema evolution.

**Ownership boundary:** This service owns all of the above exclusively. The DuckDB file is held under an exclusive lock; the comment in source explicitly warns that external processes cannot open it even read-only without hitting `DBBusyError`.

---

## Dependencies

### Runtime — External Services

| Dependency | Type | Purpose |
|---|---|---|
| Fastly CDN / Delivery API | Third-party API (HTTP) | Log delivery source; Stats API for accounting reconciliation; Compute service management (deploy scorer Wasm, manage URL-exclusion VCL) |
| FOS (Fastly Object Storage, S3-compatible) | Object storage | Durable raw log archive; Iceberg data files; source for ingest |
| Fastly KV Store | Key-value (edge) | Distributes scoring matrix to Wasm scorer instances |

### Runtime — Infrastructure

| Dependency | Type | Purpose |
|---|---|---|
| Caddy (reverse proxy) | Sidecar container | TLS termination, rate limiting, routing to frontend (:3000) and backend (:8000) |
| jemalloc (`libjemalloc2`) | System library | Memory allocator for DuckDB workloads |
| Falco binary (v2.3.0) | External binary (bundled) | VCL static analysis before publishing custom URL-exclusion regexes |

### Runtime — Notable Python Libraries

| Library | Purpose |
|---|---|
| FastAPI | REST API framework |
| DuckDB | In-process analytical query engine |
| PyIceberg (inferred) | Iceberg table management (`backend/core/iceberg`) |
| APScheduler | Background cron scheduler |
| sse-starlette | Server-Sent Events for streaming admin operations |
| `uv` | Python environment / dependency management |

### Runtime — Notable Frontend Libraries

| Library | Purpose |
|---|---|
| Next.js 16 | SSR/SSG React framework |
| TanStack Query | Server-state cache and background refetch |
| Zustand | Client UI state |
| MapLibre GL | Interactive POP location map |
| Plotly.js | Time-series and analytics charts |
| `openapi-fetch` | Type-safe API client (generated types) |
| `web-vitals` | Real User Monitoring (RUM) collection |
| `msw` | API mocking for tests |

### Build-time Only

| Dependency | Purpose |
|---|---|
| `openapi-typescript` | Generates `types/api.generated.ts` from `openapi.json` |
| `uv run python3 scripts/generate_openapi.py` | Generates `openapi.json` from FastAPI introspection |
| `xcaddy` | Builds custom Caddy binary with `caddy-ratelimit` |
| `cargo` + `wasm32-wasip1` target | Compiles Rust scorer to Wasm |
| Vitest, Playwright, Testing Library | Frontend unit and e2e testing |
| `osv-scanner` | Dependency vulnerability scanning |
| `eslint`, `knip` | Linting and dead-code detection |

---

## Deployment Model

### Container Images

| Image | Dockerfile | Base | Exposed Port |
|---|---|---|---|
| `backend` | `backend/Dockerfile` | `python:3.13-slim-bookworm` (multi-stage; builder + runner) | `8000` (internal only; accessed via Caddy or loopback) |
| `frontend` | `frontend/Dockerfile` | `node:24-slim` (multi-stage: api-schema → builder → runner) | `3000` (internal only) |
| `caddy` | `caddy/Dockerfile` | `caddy:2-builder` → `caddy:2-alpine` | `:80` (mapped to host `80:80` via backend's netns) |

**Frontend build pipeline (multi-stage):**
1. `api-schema` stage: Python 3.13 + `uv` — runs `generate_openapi.py` to emit `openapi.json` from FastAPI route introspection.
2. `builder` stage: Node 24 — `npm ci`, copies `openapi.json`, runs `openapi-typescript` to emit `types/api.generated.ts`, then `next build` (standalone output).
3. `runner` stage: Node 24 slim — copies only `public/`, `.next/standalone/`, `.next/static/`; runs as non-root `nextjs:nodejs (uid/gid 1001)`.

**Backend build pipeline (multi-stage):**
1. `builder` stage: installs Python deps into `.venv` via `uv sync --no-dev --frozen` with BuildKit cache mount.
2. `runner` stage: copies `.venv`, backend source, `generate_openapi.py`, and `matrix.default.json`; installs `curl`, `tar`, `libjemalloc2`, and Falco binary; runs as non-root `app:app (uid/gid 1000)`.

### Orchestration

| Environment | Tooling | Notes |
|---|---|---|
| Local development | `docker-compose.yml` | All three containers share backend's network namespace (`network_mode: "service:backend"`); port `80:80` published by backend service |
| Production | `docker-compose.prod.yml` (referenced but not provided) | Overrides to `network_mode: host`; custom Caddy image with `caddy-ratelimit` |
| CI | `.github/` workflows (`ci.yml`, `e2e.yml`, `perf-nightly.yml`, `cidr-refresh.yml`) | Gates: ESLint ceiling, OSV scan, security regression count, OTel console guard, perf gate |

### Volumes (docker-compose)

| Host path | Container path | Purpose |
|---|---|---|
| `./configs` | `/app/configs` | Service configuration YAML/JSON |
| `./data` | `/app/data` | DuckDB files, SQLite DBs |
| `./cache` | `/app/cache` | Local Parquet cache |

### Environment Configuration

| Variable | Default | Purpose |
|---|---|---|
| `PYTHONUNBUFFERED` | `1` | Unbuffered stdout/stderr |
| `DUCKDB_MEMORY_LIMIT` | `4GB` | DuckDB memory cap |
| `ARROW_DEFAULT_MEMORY_POOL` | `system` | Arrow allocator routing to jemalloc |
| `LOCAL_HOSTS` | `fastly.localhost,fastly.analytics` | Additional hostnames added to backend loopback allowlist |
| `API_PROXY_URL` | `http://127.0.0.1:8000` | Frontend SSR proxy target (must be loopback, not service DNS) |
| `SCORING_REQUIRE_FALCO` | `1` (production) | Hard-fail VCL validator if Falco binary is absent |
| `NEXT_TELEMETRY_DISABLED` | `1` | Disables Next.js telemetry |
| `PORT` / `HOSTNAME` | `3000` / `0.0.0.0` | Next.js listener config |

### Health / Readiness

| Service | Endpoint | Interval | Timeout | Retries | Start period |
|---|---|---|---|---|---|
| backend | `GET http://localhost:8000/api/health` (curl) | 30 s | 5 s | 3 | 15 s |
| frontend | _Not determinable from code._ | — | — | — | — |
| caddy | _Not determinable from code._ | — | — | — | — |

**Admin health snapshot** is also available at `GET /admin/health-snapshot` (load averages, memory, disk, DuckDB pool wait stats, in-flight cron runs, scheduler liveness, config-backup freshness, FOS reachability probe opt-in).
