---
repo: fastly-log-analytics
spec_type: behavioral
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:58:31.107203435+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST/HTTP (FastAPI backend, served via Caddy reverse proxy). All backend routes are prefixed with `/api`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/api/admin/bot-sources` | Return metadata for all bot sources plus rDNS cache stats | — | `BotSourcesResponse` |
| POST | `/api/admin/bot-sources/{source_id}/refresh` | Fetch and re-cache a single bot source | Path: `source_id: str` | `{"ok": true, "source": <meta>}` or 404/502 |
| POST | `/api/admin/optimize-now` | Trigger immediate Iceberg table compaction (writes to FOS) | Query: `min_files?: int`; `source` (dep) | Optimize result dict (`files_rewritten`, `files_added`, etc.) |
| POST | `/api/admin/backfill-bundle-rollups` | One-shot rebuild of missing hourly/daily rollup bundles | `source` (dep) | `{"slow_urls": int, "origin_summary": int, "origin_summary_days": int, "network_rtt": int, "network_rtt_days": int, "network_speed": int, "network_speed_days": int, "verified_bots_ts": int, "verified_bots_ts_days": int, "perf_latency": int, "perf_latency_days": int}` |
| POST | `/api/admin/local-compact-now` | Trigger local-only parquet compaction (no FOS writes) | Query: `min_files: int = 3`, `dry_run: bool = false`; `source` (dep) | Compaction result |
| GET | `/api/admin/compaction-stats` | Snapshot of file-count distribution across local cache partitions | `source` (dep) | `CompactionStatsResponse` |
| PATCH | `/api/admin/metadata-retention` | Update per-service metadata retention config | Body: `{usage_log_days?, ingested_files_days?, cron_runs_days?}`; `source` (dep) | _Not determinable from code._ |
| GET | `/api/admin/metadata-storage` | _Not determinable from code._ | `source` (dep) | _Not determinable from code._ |
| POST | `/api/admin/metadata-cleanup` | _Not determinable from code._ | `source` (dep) | _Not determinable from code._ |
| GET | `/api/admin/health-snapshot` | System health snapshot (CPU, memory, disk, cron status, pool wait, backup freshness) | Query: `probe_fos: bool = false` | `HealthSnapshotResponse` (load, vcpus, memory, data_mount, root_disk, in_flight_runs, compaction, pool_wait, scheduler liveness, etc.) |
| GET | `/api/admin/iceberg-info` | Iceberg table metadata: snapshots, data files, size, buffer status | `source` (dep) | `IcebergTableInfoResponse` |
| GET | `/api/admin/iceberg-calendar` | Per-date data file counts from Iceberg partition metadata | `source` (dep) | `{<calendar data>, "_debug_calls": [...]}` |
| POST | `/api/admin/commit-iceberg` | Manually flush local buffer to Iceberg table (async) | `source` (dep) | 202: `{"ok": true, "message": str, "run_id": str}` |
| POST | `/api/admin/rebuild-local-view` | Clear caches and rebuild local DuckDB view from cloud catalog (async) | `source` (dep) | 202: `{"ok": true, "message": str, "run_id": str}` |
| POST | `/api/admin/ingest-logs` | Manually trigger log ingestion or metadata sync | Query: `start_time?: str`, `end_time?: str`; `source` (dep) | `{"ok": true, "message": str, "run_id": str}` |
| POST | `/api/admin/backfill-window` | Force-sync a specific time window from FOS into local cache | Query: `start_time: str (ISO 8601)`, `end_time: str (ISO 8601)`; `source` (dep) | `dict` |
| GET | `/api/admin/log-accounting` | Fastly Stats vs locally-ingested counts comparison | `source` (dep); additional query params _not fully determinable_ | `LogAccountingResponse` |
| GET | `/api/admin/metric-history/batch` | Batch metric history retrieval | _Not determinable from code._ | _Not determinable from code._ |
| GET | `/api/admin/pop-locations` | Return Fastly PoP location data | — | _Not determinable from code._ |
| POST | `/api/admin/pop-locations/refresh` | Refresh cached PoP location data | — | _Not determinable from code._ |
| GET | `/api/download-folder` | Stream a ZIP of a folder from FOS/CDN | Query: `prefix: str = ""`, `root: str = "raw"`; `source` (dep) | `StreamingResponse` (`application/zip`), `Content-Disposition: attachment; filename="<name>.zip"` |
| GET | `/api/download` | Download a single file (local cache hit → FileResponse; CDN → proxied stream; FOS → direct) | Query: `key: str`; `source` (dep) | `FileResponse` or `StreamingResponse`; 400 on missing/invalid key |
| GET | `/api/download-all` | Download all files for a service as a ZIP | `source` (dep) | `StreamingResponse` (`application/zip`) |
| GET | `/api/health` | Liveness/readiness healthcheck (used by Docker healthcheck) | — | _Not determinable from code._ |

**`source` dependency** (`get_source`): resolved from request context; provides `service_id`, `name`, `bucket`, `prefix`, `cdn_url`, `cdn_secret`, `access_level`, and related config fields.

**Edge compute (Fastly Compute @ Wasm):** The `session-scorer` binary (`compute/scorer/`) runs on Fastly's edge. It exposes no REST API directly; it intercepts HTTP requests at the edge, scores sessions, and emits an `X-Edge-Sid` response header containing a hex-encoded session ID. It reads a scoring matrix from Fastly KV Store (binary FSM1 format) and manages session cookies using AES-GCM authenticated encryption.

---

## Event Schemas

_Not determinable from code._

No message broker integration (Kafka, RabbitMQ, etc.) is evidenced in the source snapshot, dependency manifests, or extracted facts. The service uses internal APScheduler cron jobs for background work (ingest, compaction, commit, metadata sync), but these are in-process and not exposed as external events or topics.

---

## Input / Output Formats

**Serialization:** JSON is the primary serialization format for all REST request and response bodies (FastAPI default). Binary formats are used in specific contexts:
- **Parquet** files for local analytics data cache.
- **FSM1** (hand-rolled binary format) for the edge scorer's matrix, loaded from Fastly KV Store. No serde/serde_json dependency is used in the Wasm binary to minimize cold-start size.
- **ZIP (Deflate)** for the download-folder and download-all streaming endpoints (`application/zip`).
- **Base64** encoding used within the edge scorer (cookie wire format).
- **Avro/Protobuf:** _Not determinable from code._

**Content types observed:**
- `application/json` — standard API request/response bodies.
- `application/zip` — streaming download endpoints.
- `application/octet-stream` — fallback for single-file CDN proxy downloads.
- `text/event-stream` — SSE (`EventSourceResponse` from `sse-starlette`) is imported in `compaction.py`; at least one endpoint emits server-sent events (exact endpoint _not fully determinable from code_).

**Frontend API client:** The frontend uses `openapi-fetch` (typed against auto-generated `types/api.generated.ts` from the OpenAPI schema) for all backend calls. TypeScript types are regenerated at build time from `scripts/generate_openapi.py` + `openapi-typescript`.

**Pagination:** _Not determinable from code._ No pagination envelope is evidenced in the extracted endpoint contracts.

**Request envelopes:** No standard envelope wrapper is used; payloads are direct Pydantic model bodies or query parameters.

**Response envelopes:** Responses with telemetry use a `.with_telemetry(...)` constructor pattern (e.g., `BotSourcesResponse.with_telemetry(...)`, `IcebergTableInfoResponse.with_telemetry(...)`). Async-start endpoints return `{"ok": true, "message": str, "run_id": str}`.

---

## Error Handling

**Error model:** FastAPI's default HTTP exception model, supplemented by a shared `DEFAULT_ERROR_RESPONSES` dict from `backend.models.errors` applied to the admin `APIRouter`.

**Evidenced status codes and mappings:**

| Status Code | Trigger |
|-------------|---------|
| 400 | Missing or invalid `key` query parameter on `/api/download`; key outside service prefix (cross-tenant guard); path-traversal detected |
| 404 | Bot source not found (`ValueError` from `fetch_and_cache_source` → `HTTPException(404, not_found("bot_source_not_found"))`) |
| 202 | Async cron-start responses (`/admin/commit-iceberg`, `/admin/rebuild-local-view`, `/admin/ingest-logs`) |
| 500 | Internal errors via `raise_internal(logger, exc, code=..., status=500)`; `@query_errors(status_code=500)` decorator on several endpoints |
| 502 | CDN/FOS fetch failures (`raise_internal(..., status=502)`); bot source fetch failure (`raise_internal(..., code="bot_source_fetch_failed", status=502)`) |
| 503 | Cron busy on `/admin/rebuild-local-view` (`HTTPException(503, make_error("cron_busy", ..., busy=True))`) |

**Error payload structure:** Errors use a structured dict:
```json
{"error": "<code>", "detail": "<message>"}
```
or the `make_error(code, message, **kwargs)` helper which can include additional fields (e.g., `busy: true` for 503 cron-busy responses).

**Validation:** FastAPI/Pydantic model validation applies to all typed request bodies. Query parameter validation is also Pydantic-enforced (e.g., `min_files: int`, `ge=0` constraint on local-compact-now).

**Streaming error handling:** `ClientDisconnected` exception class is used to abort streaming ZIP generation when the client disconnects mid-stream; it is swallowed by the `except Exception` in the ZIP worker to keep the response stream coherent.

**Cross-tenant guard:** `/api/download` enforces that the requested key falls under the service's configured prefix, returning 400 (`invalid_key`) otherwise.

---

## Versioning

**URI versioning:** Not used. All admin API routes are prefixed with `/api` (no `/v1/`, `/v2/` segment).

**Application version:** `frontend/package.json` declares `"version": "2.0.0-beta.1"`. The `backend/Dockerfile` does not embed a version header in responses.

**Schema evolution:** The OpenAPI schema is generated deterministically at build time (`scripts/generate_openapi.py`) and TypeScript types are regenerated from it (`scripts/refresh_api_types.js` / `openapi-typescript`). This constitutes a code-generation–based schema-sync strategy rather than a runtime versioning header or content-negotiation approach. A `regen-openapi` pre-push git hook and CI gate enforce schema freshness.

**No API version header or `Accept` content-negotiation versioning** is evidenced in the source.
