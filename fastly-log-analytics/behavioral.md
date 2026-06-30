---
repo: fastly-log-analytics
spec_type: behavioral
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:57:16.978197709+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP (FastAPI/Python backend). All routes are prefixed with `/api`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| GET | `/api/admin/bot-sources` | Return metadata for all bot sources plus rDNS cache stats | — | `BotSourcesResponse` |
| POST | `/api/admin/bot-sources/{source_id}/refresh` | Fetch and re-cache a single bot source | Path: `source_id` (str) | `{"ok": true, "source": <meta>}` or 404/502 |
| POST | `/api/admin/optimize-now` | Trigger immediate Iceberg table compaction (writes to FOS) | Query: `min_files` (int, optional) | Optimize result dict (`files_rewritten`, `files_added`, etc.) |
| POST | `/api/admin/backfill-bundle-rollups` | One-shot self-heal for slow_urls + origin_summary rollups | — | `{"slow_urls": N, "origin_summary": N, "origin_summary_days": N, "network_rtt": N, ...}` |
| POST | `/api/admin/local-compact-now` | Immediate local-only parquet compaction (no FOS writes) | Query: `min_files` (int, default 3), `dry_run` (bool, default false) | Compaction result dict |
| GET | `/api/admin/compaction-stats` | Snapshot of file-count distribution across local cache partitions | — | `CompactionStatsResponse` |
| PATCH | `/api/admin/metadata-retention` | Update per-service metadata retention config | JSON body: subset of `{usage_log_days, ingested_files_days, cron_runs_days}` | _Not determinable from code._ |
| GET | `/api/admin/metadata-storage` | _Not determinable from code._ | — | `metadata_retention`, `ingested_files_days` fields |
| POST | `/api/admin/metadata-cleanup` | _Not determinable from code._ | — | _Not determinable from code._ |
| GET | `/api/download-folder` | Stream a ZIP of objects under an FOS prefix | Query: `prefix` (str, default ""), `root` (str, default "raw") | `StreamingResponse` (application/zip) |
| GET | `/api/download` | Download a single file (local cache, CDN, or FOS) | Query: `key` (str) | `FileResponse` or `StreamingResponse` with Content-Type/Content-Length headers, or 400/502 |
| GET | `/api/download-all` | _Not determinable from code._ | — | _Not determinable from code._ |
| GET | `/api/admin/health-snapshot` | System health snapshot (CPU, memory, disk, cron state, compaction) | Query: `probe_fos` (bool, default false) | `HealthSnapshotResponse` |
| GET | `/api/admin/iceberg-info` | Iceberg table metadata: snapshots, data files, size, buffer status | — | `IcebergTableInfoResponse` |
| GET | `/api/admin/iceberg-calendar` | Per-date data file counts from Iceberg partition metadata | — | `{...result, "_debug_calls": [...]}` |
| POST | `/api/admin/commit-iceberg` | Manually flush local buffer to Iceberg table (async) | — | `{"ok": true, "message": ..., "run_id": ...}` — HTTP 202 |
| POST | `/api/admin/rebuild-local-view` | Clear caches and trigger metadata sync to rebuild local DuckDB view (async) | — | `{"ok": true, "message": ..., "run_id": ...}` — HTTP 202, or 503 if cron busy |
| POST | `/api/admin/ingest-logs` | Manually trigger log ingestion or metadata sync | Query: `start_time` (str, optional), `end_time` (str, optional) | `start_or_resume_cron` response dict with `run_id` |
| POST | `/api/admin/backfill-window` | Force-sync a specific time window from FOS into local cache | Query: `start_time` (ISO 8601, required), `end_time` (ISO 8601, required) | _Not determinable from code._ |
| GET | `/api/admin/log-accounting` | Fastly Stats vs locally-ingested counts | _Not determinable from code._ | `LogAccountingResponse` |
| GET | `/api/admin/metric-history/batch` | _Not determinable from code._ | — | _Not determinable from code._ |
| GET | `/api/admin/pop-locations` | _Not determinable from code._ | — | _Not determinable from code._ |
| POST | `/api/admin/pop-locations/refresh` | Refresh PoP location data | — | _Not determinable from code._ |
| GET | `/api/health` | Health check (used by Docker healthcheck) | — | _Not determinable from code._ |

The backend also exposes a `/api/bootstrap` endpoint (referenced in `compute_sync_status_cached`). All admin endpoints require a resolved `source` dependency (`get_source`) that provides service configuration including `service_id`, `name`, `bucket`, `prefix`, `cdn_url`, `cdn_secret`, and `access_level`.

**Edge / Compute component:** A Rust/Wasm binary (`session-scorer`) runs on Fastly Compute@Edge. It exposes no HTTP API surface of its own — it is deployed to the Fastly edge and intercepts/annotates requests, emitting an `X-Edge-Sid` response header (hex-encoded session ID bytes).

## Event Schemas

_Not determinable from code._

No Kafka, RabbitMQ, or other message broker topics/queues were detected in the extracted facts or source samples. Asynchronous work is handled internally via APScheduler cron jobs and Python `threading.Thread` daemon threads, not via an external message bus.

## Input / Output Formats

**Serialization:** JSON for all REST request and response bodies. File download endpoints return `application/zip` (streaming ZIP via `StreamingResponse`) or binary file content (`FileResponse`).

**Content types observed:**
- Request bodies: `application/json`
- Standard API responses: `application/json`
- Download endpoints: `application/zip` (folder/all downloads), `application/octet-stream` or passthrough `Content-Type` from CDN (single file download)
- The `/admin/health-snapshot` and `/admin/iceberg-calendar` endpoints return plain JSON dicts (not Pydantic response models in all cases)

**OpenAPI schema generation:** The build pipeline runs `scripts/generate_openapi.py` to emit `openapi.json` from FastAPI route introspection, then `scripts/refresh_api_types.js` runs `openapi-typescript` to generate `types/api.generated.ts` for the frontend. The frontend consumes the backend via the `openapi-fetch` client library typed against this generated schema.

**Pagination:** _Not determinable from code._

**Request envelopes:** No wrapper envelope is used on requests; parameters are passed as query strings or JSON body fields directly. Responses from async-trigger endpoints follow the pattern `{"ok": true, "message": "...", "run_id": "..."}`.

**Streaming responses:** `StreamingResponse` is used for ZIP downloads; content is yielded from a daemon thread via a `queue.Queue`-backed `_QueueFile` with an `_AbortableQueue` for client-disconnect signalling.

**Time format:** ISO 8601 UTC strings are used for time parameters (e.g., `start_time`, `end_time`) and in response fields such as `started_at`.

**Edge scorer binary format:** The session-scorer Wasm reads its scoring matrix from the Fastly KV Store in a hand-rolled binary format (`FSM1`); no JSON/serde involved. Session cookie values use base64 encoding; session IDs are hex-encoded in the `X-Edge-Sid` response header.

## Error Handling

**Error model:** The shared `APIRouter` is instantiated with `responses=DEFAULT_ERROR_RESPONSES` (from `backend.models.errors`), applying a standard error response schema to all admin endpoints. Specific evidenced mappings:

| Condition | HTTP Status | Response body shape |
|-----------|-------------|---------------------|
| Bot source not found | 404 | `not_found("bot_source_not_found")` detail |
| Bot source fetch failure | 502 | `{"error": "bot_source_fetch_failed", ...}` |
| Missing `key` query param on `/download` | 400 | `{"error": "Missing key parameter"}` |
| Path traversal / cross-tenant key on `/download` | 400 | `{"error": "invalid_key"}` |
| CDN fetch failure on `/download` | 502 | `{"error": "cdn_fetch_failed", ...}` |
| Cron already busy on `/admin/rebuild-local-view` | 503 | `{"error": "cron_busy", ..., "busy": true}` |
| Snapshot cache file removal failure | 500 | `{"error": "snapshot_cache_remove_failed"}` |
| Generic internal errors | 500 | `raise_internal(logger, exc, code=<str>, status=500)` pattern |

**`@query_errors` decorator:** Applied to several endpoints (e.g., `/admin/iceberg-info`, `/admin/iceberg-calendar`, `/download`) — catches query-layer exceptions and maps them to HTTP 500 responses.

**`raise_internal` helper:** Used throughout to log at ERROR level and raise `HTTPException` with a structured `{"error": <code>, ...}` detail.

**Validation:** FastAPI's built-in Pydantic validation applies to all declared request/response models. Body fields for PATCH `/admin/metadata-retention` are coerced to `int`; negative values handling is noted in the source but exact behaviour is not fully determinable from the visible code.

**Async trigger endpoints:** `/admin/commit-iceberg` and `/admin/rebuild-local-view` return HTTP 202 rather than 200 to prevent clients from treating an async-started operation as complete.

## Versioning

No URI versioning prefix (e.g., `/v1/`, `/v2/`) or API version header strategy is evident in the extracted routes or source samples. All routes use `/api/admin/...` or `/api/...` flat paths.

The `frontend/package.json` declares `"version": "2.0.0-beta.1"` and the package name is `fastly-log-analysis-frontend`, indicating a major version transition at the product level, but this is not reflected in the API route structure.

The OpenAPI schema is regenerated at build time from FastAPI route definitions (`scripts/generate_openapi.py`); schema evolution is managed via `openapi-typescript` type regeneration rather than a versioned schema registry. No `Accept-Version` header, content negotiation, or schema evolution strategy (e.g., Avro schema registry) is evidenced.
