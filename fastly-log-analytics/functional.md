---
repo: fastly-log-analytics
spec_type: functional
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:58:31.107203435+02:00
generator: specsync
---

## Business Purpose

`fastly-log-analytics` is a self-hosted analytics platform that ingests, stores, and queries Fastly CDN access logs for a single operator/organisation. It provides a full-stack dashboard (Next.js frontend + Python/FastAPI backend) over locally-cached Parquet/Iceberg data, enabling operators to analyse traffic, performance, bot activity, geographic distribution, and origin health without sending raw log data to a third-party analytics service. An optional Rust/Wasm edge component (`session-scorer`) runs on Fastly Compute to classify and score sessions at the CDN edge before logs are written.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** CDN Log Analytics — the ownership boundary covers everything from raw Fastly log ingestion through local query, rollup, and presentation layers.
- **Core domain entities / aggregates:**
  - **Service** — a Fastly CDN service configuration (identified by `service_id` / `name`), the root aggregate for all data scoped below.
  - **Log file** — raw compressed log objects stored in Fastly Object Storage (FOS/S3-compatible).
  - **Iceberg table / local Parquet cache** — the queryable materialisation of ingested logs.
  - **Rollup bundles** — pre-aggregated hourly/daily summaries (slow URLs, origin summary, network RTT/speed, verified bots, perf latency).
  - **Session** — an edge-classified visitor session (scored and identified by the Wasm edge component).
  - **Bot source** — a catalogued source of bot IP ranges with rDNS metadata.
  - **Cron run** — a tracked execution of a scheduled or manual background task (ingest, commit, metadata sync, compaction).
  - **POP location** — Fastly point-of-presence geographic metadata.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Fastly CDN platform (provides raw logs via FOS object storage, stats API, VCL/Compute runtime, and POP location data).
  - **Upstream:** Fastly Stats API (authoritative request counts used for log-accounting/loss detection).
  - **Downstream:** The edge `session-scorer` Wasm binary is published to the customer's Fastly Compute service via the Fastly API; it feeds session classification data back into the log stream.

## Use Cases / User Stories

- **As an operator, I want to manually trigger log ingestion for a time window** so that I can fill gaps or force a refresh without waiting for the cron. → `POST /admin/ingest-logs`
- **As an operator, I want to view a health snapshot of the running system** so that I can detect resource exhaustion, stalled crons, or failed scheduler tasks. → `GET /admin/health-snapshot`
- **As an operator, I want to compare locally ingested log counts against Fastly Stats API counts** so that I can detect and diagnose sustained log loss. → `GET /admin/log-accounting`
- **As an operator, I want to force-sync a specific historical time window** so that I can heal gaps identified by the log-accounting view. → `POST /admin/backfill-window`
- **As an operator, I want to trigger Iceberg table compaction** so that I can reclaim storage and improve query performance outside of the nightly cron schedule. → `POST /admin/optimize-now`, `POST /admin/local-compact-now`
- **As an operator, I want to rebuild rollup bundles for historical hours** so that I can repair missing aggregations without a full re-ingest. → `POST /admin/backfill-bundle-rollups`
- **As an operator, I want to rebuild the local DuckDB view** so that I can recover from a desynchronised or stale query layer. → `POST /admin/rebuild-local-view`
- **As an operator, I want to manually flush the local write buffer to the Iceberg table** so that I can ensure recent data is durably committed. → `POST /admin/commit-iceberg`
- **As an operator, I want to view and refresh bot source lists** so that bot traffic is correctly classified in analytics. → `GET /admin/bot-sources`, `POST /admin/bot-sources/{source_id}/refresh`
- **As an operator, I want to view Iceberg table metadata and a per-date data file calendar** so that I can verify data coverage and table health. → `GET /admin/iceberg-info`, `GET /admin/iceberg-calendar`
- **As an operator, I want to download individual log files or batches as ZIP archives** so that I can perform offline analysis or archival. → `GET /download`, `GET /download-folder`, `GET /download-all`
- **As an operator, I want to view and update metadata retention configuration** so that I can control storage costs. → `PATCH /admin/metadata-retention`, `GET /admin/metadata-storage`, `POST /admin/metadata-cleanup`
- **As an operator, I want to view batch metric history** so that I can track system performance over time. → `GET /admin/metric-history/batch`
- **As an operator, I want to view compaction statistics** so that I can monitor whether the compaction cron is keeping up with write volume. → `GET /admin/compaction-stats`
- **As an operator, I want to view and refresh POP location data** so that geographic analytics are accurate. → `GET /admin/pop-locations`, `POST /admin/pop-locations/refresh`
- **As an operator, I want to trigger an ad-hoc log-accounting backfill for a specific window** so that I can resolve counted gaps retroactively. → `POST /admin/backfill-window`
- **As an operator, I want to trigger or schedule edge session scoring** so that bot/anomaly detection runs at the CDN edge with minimal latency. → Edge `session-scorer` Wasm deployed via `scripts/scoring/deploy_wasm.sh`
- **As a developer, I want to run synthetic load tests and perf gates** so that performance regressions are caught before deployment. → `scripts/perf_gate.sh`, `scripts/emit_perf_latest.py`

## Business Rules

- **Local admin trust model:** Requests arriving on loopback (`127.0.0.1`) are treated as local/admin; requests from non-loopback addresses are classified as remote and may be rejected (400). The Caddy/frontend containers deliberately share the backend's network namespace to satisfy this rule. (inferred from `backend/utils/remote_access.py` references and `docker-compose.yml` comments)
- **Cross-tenant key isolation:** File download and presigned URL minting require that the requested object key begins with the service's configured `prefix`. Requests for keys outside the prefix are rejected with HTTP 400 `invalid_key`.
- **Path traversal prevention:** Local cache file serving resolves the candidate path and requires it to be within the cache directory; absolute paths or `../` traversals are rejected with HTTP 400 `invalid_key`.
- **CDN secret confidentiality:** The `cdn_secret` bearer token is never embedded in redirect URLs; the backend fetches CDN objects server-side using an `x-fastly-key` header so the secret never leaves the service trust boundary.
- **Read-only service access level:** If a service is configured with `access_level = read_only`, manual ingest triggers a `metadata_sync` rather than a full `_run_service_cron` write path.
- **Cron concurrency guard:** Only one instance of a given cron task (e.g. `commit`, `metadata_sync`) may run per service at a time; a second request returns HTTP 503 `cron_busy` (for `rebuild-local-view`) or resumes/reports the existing run_id (for other endpoints).
- **Negative metadata retention values are rejected:** Values passed to `PATCH /admin/metadata-retention` are coerced to int; negative values are explicitly disallowed. (inferred from code comment "negative …")
- **Iceberg optimize minimum file threshold:** The compaction endpoint accepts an optional `min_files` override; passing `1` enables maximum-aggressive cleanup, `0` force-rewrites every partition. Default cron behaviour uses `min_files = 3`.
- **`local-compact-now` is FOS-safe:** Local compaction only rewrites files in the local cache and never touches Fastly Object Storage, avoiding object-storage minimum-storage-duration billing penalties.
- **Wasm scorer cold-start size constraint:** The `session-scorer` Cargo profile uses `opt-level = "s"`, LTO, single codegen unit, symbol stripping, and `panic = abort` to minimise `.wasm` binary size for fast per-request instantiation on Fastly Compute.
- **Falco VCL validator is required in production:** If `SCORING_REQUIRE_FALCO=1` is set, absence of the `falco` binary causes a hard failure when publishing a custom URL-exclusion regex. (inferred from Dockerfile comment)
- **Directory size cache TTL is 30 seconds:** Stale-while-revalidate with background refresh; concurrent cold misses are coalesced to a single walk per path.
- **Log-accounting Fastly Stats cache TTL is 45 seconds:** The dominant latency item (Fastly Stats API fetch, ~1.8 s p95) is memoised per (service, from_ts, to_ts, by) key for 45 s to avoid visible spinner latency on the admin polling interval.
- **Session ID generation requires real entropy:** Without `getrandom`, all cookie-less visitors collapse to a single session ID, breaking per-visitor analytics. The scorer explicitly depends on `getrandom = "0.2"` for WASI random entropy. (inferred)
- **Config backup freshness is monitored:** The admin health card reads a freshness marker written by `scripts/backup_service_configs.sh`; a missing or stale marker surfaces as a health warning. (inferred from `CONFIG_BACKUP_MARKER` endpoint and Dockerfile/script references)
- **OSV vulnerability scanning is a CI gate:** `scripts/check_osv.py` runs `osv-scanner` and fails the build if vulnerability severity exceeds a configured threshold.
- **Security regression test count is a CI floor:** `scripts/check_security_regression_count.sh` enforces a minimum number of security regression tests; the count may not decrease below the current floor.
