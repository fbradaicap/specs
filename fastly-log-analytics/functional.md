---
repo: fastly-log-analytics
spec_type: functional
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:57:16.978197709+02:00
generator: specsync
---

## Business Purpose

`fastly-log-analytics` is a self-hosted analytics platform for Fastly CDN customers. It ingests raw Fastly access logs from object storage (FOS/S3-compatible), stores and compacts them locally using DuckDB and Apache Iceberg, and exposes a Next.js dashboard with charts, maps, SQL querying, and bot-detection insights. The service exists so operators can analyse CDN traffic patterns, monitor origin performance, detect anomalies, and reconcile log accounting against Fastly's own stats API — without routing data through a third-party SaaS.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** CDN Log Analytics & Session Intelligence — covering ingestion, storage, querying, and visualisation of Fastly CDN access-log data.
- **Core domain entities / aggregates:**
  - **Service** — a Fastly CDN service configuration (service_id, bucket, prefix, CDN URL/secret, access level, metadata-retention policy).
  - **Log File / Ingest Record** — raw `.gz` Fastly log objects in object storage, tracked by ingestion state (in-flight, ingested, orphaned).
  - **Iceberg Table / Snapshot** — the primary analytical store; versioned table of parsed log events partitioned by date.
  - **Local Parquet Cache** — compacted local DuckDB-backed cache of Iceberg data for fast dashboard queries.
  - **Rollup Bundle** — pre-aggregated per-hour and per-day summaries (slow URLs, origin summary, network RTT/speed, verified bots, perf latency).
  - **Session** — a reconstructed visitor session; scored by the edge Wasm scorer for anomaly/bot signals.
  - **Bot Source** — a named external feed of known-bot IP ranges / rDNS patterns.
  - **PoP Location** — Fastly Point of Presence geo-coordinate record.
- **Relationships to neighbouring contexts:**
  - **Upstream — Fastly CDN platform:** log files are produced by Fastly logging endpoints into object storage; Fastly Stats API is called to validate local counts; Fastly Compute API is used to deploy/manage the edge session-scorer Wasm.
  - **Upstream — Object storage (FOS/S3-compatible):** raw log source and Iceberg table backing store.
  - **Downstream — Browser dashboard (Next.js frontend):** consumes the backend REST API for all analytics, admin, and download operations.
  - **Downstream — Fastly Compute@Edge:** the compiled `session-scorer` Wasm is published to the customer's Compute service to score sessions at the edge.

## Use Cases / User Stories

- **As an operator, I want to trigger manual log ingestion for a time window** so that I can fill gaps caused by missed cron runs. → `POST /admin/ingest-logs?start_time=…&end_time=…`
- **As an operator, I want to view a log accounting comparison** (local ingest counts vs Fastly Stats API) so that I can detect classifier drift or mid-flight backfill anomalies. → `GET /admin/log-accounting`
- **As an operator, I want to force-backfill a specific time window from FOS** so that I can repair gaps in the local cache. → `POST /admin/backfill-window`
- **As an operator, I want to trigger compaction / optimization of the Iceberg table** so that I can reduce file count and keep query performance healthy. → `POST /admin/optimize-now`, `POST /admin/local-compact-now`
- **As an operator, I want to view compaction statistics** so that I can monitor whether the local compaction cron is keeping up. → `GET /admin/compaction-stats`
- **As an operator, I want to backfill missing rollup bundles** (slow URLs, origin summary, network RTT/speed, verified bots, perf latency) for historical hours so that the dashboard panels have complete data. → `POST /admin/backfill-bundle-rollups`
- **As an operator, I want to view Iceberg table metadata and a per-date snapshot calendar** so that I can verify data completeness. → `GET /admin/iceberg-info`, `GET /admin/iceberg-calendar`
- **As an operator, I want to manually flush the local buffer to the Iceberg table** so that uncommitted data is persisted immediately. → `POST /admin/commit-iceberg`
- **As an operator, I want to rebuild the local DuckDB view** so that I can recover from a stale or desynchronised analytics view. → `POST /admin/rebuild-local-view`
- **As an operator, I want to download individual log files or entire folders as ZIP archives** so that I can retain or process raw data externally. → `GET /download`, `GET /download-folder`, `GET /download-all`
- **As an operator, I want to see a system health snapshot** (CPU, memory, disk, pool wait, scheduler liveness, cron errors, config-backup freshness) so that I can diagnose operational issues. → `GET /admin/health-snapshot`
- **As an operator, I want to update metadata-retention policy** so that I can control how long usage logs and ingested file records are kept. → `PATCH /admin/metadata-retention`
- **As an operator, I want to trigger a metadata cleanup** so that I can reclaim disk space by purging expired records. → `POST /admin/metadata-cleanup`
- **As an operator, I want to inspect and refresh bot-source feeds** so that bot classification uses up-to-date IP/rDNS lists. → `GET /admin/bot-sources`, `POST /admin/bot-sources/{source_id}/refresh`
- **As an operator, I want to view and refresh PoP location data** so that the geographic traffic map is accurate. → `GET /admin/pop-locations`, `POST /admin/pop-locations/refresh`
- **As an operator, I want to view metric history in batch** so that I can plot historical performance trends. → `GET /admin/metric-history/batch`
- **As an operator, I want to view metadata storage statistics** so that I can understand per-service storage consumption. → `GET /admin/metadata-storage`
- **As a dashboard user, I want to query CDN log data using SQL** so that I can perform ad-hoc analysis beyond pre-built panels.
- **As a developer, I want to train and deploy a session-anomaly scoring Wasm to Fastly Compute** so that bot and anomaly signals are evaluated at the edge before logs reach the backend. → `scripts/scoring/train.py`, `scripts/scoring/deploy_wasm.sh`

## Business Rules

- **Access-level enforcement:** Services configured with `access_level == "read_only"` only trigger a metadata sync (not a full ingest cron) when `POST /admin/ingest-logs` is called; full ingest is blocked for read-only sources. (inferred)
- **Cross-tenant key isolation:** A log file download key must begin with the configured per-service `prefix`; requests for keys outside that prefix are rejected with HTTP 400 `invalid_key`. This prevents one service's configuration from accessing another tenant's FOS objects.
- **Path-traversal prevention:** Local cache file downloads are bounds-checked against the service cache directory using real-path resolution; absolute paths and `../` traversal payloads are rejected with HTTP 400 `invalid_key`.
- **CDN secret confidentiality:** When proxying CDN downloads, the CDN secret is transmitted as an HTTP request header (`x-fastly-key`) server-side, never embedded in redirect URLs, to prevent leaking the bearer token into browser history, Referer headers, or HTTP intermediaries.
- **Compaction cron coalescing:** Only one compaction/commit/metadata-sync run per service may be active at a time; a second trigger returns the existing `run_id` (resume) or HTTP 503 `cron_busy` (rebuild endpoint), preventing concurrent DuckDB exclusive-lock conflicts.
- **DuckDB exclusive-lock requirement:** Backfill and rollup endpoints must be called via the running backend process; an external process cannot access the `.duckdb` file because DuckDB holds an exclusive lock.
- **Metadata-retention coercion:** Retention values in `PATCH /admin/metadata-retention` are coerced to `int`; negative values are not accepted. (inferred)
- **Iceberg optimize minimum-files threshold:** `POST /admin/optimize-now` accepts an optional `min_files` override; without it an auto-derived threshold is used. The `POST /admin/local-compact-now` endpoint defaults to `min_files=3` (normal cron behaviour) and accepts `0` to force-rewrite every partition.
- **Config-backup freshness monitoring:** The admin health card reads a freshness marker written by `scripts/backup_service_configs.sh`; a missing or stale marker is surfaced as a health signal (SRE-11).
- **Log-accounting loss alerting:** There is a named threshold `LOG_ACCOUNTING_LOSS_THRESHOLD` and a minimum run count `LOG_ACCOUNTING_MIN_RUN` used by both the UI callout and the gap-heal cron to trigger `SustainedLossAlert` when local ingest counts fall below expected Fastly Stats API values.
- **Falco VCL validation enforcement:** When `SCORING_REQUIRE_FALCO=1` is set on the backend, the VCL validator hard-fails if the `falco` binary is absent, preventing deployment of an unvalidated URL-exclusion regex to the scoring service.
- **Remote vs. local trust boundary:** Requests arriving from loopback (127.0.0.1) are classified as local admin; requests from non-loopback addresses are classified as remote and subject to additional access restrictions (`backend/utils/remote_access.py`).
- **OTel console exporter prohibited in deployed environments:** CI enforces via `check_no_console_otel.sh` that `OTEL_EXPORTER=console` is never set in deployed configuration (ADR-08).
- **Security regression floor:** The number of security-regression tests may not decrease; enforced by `check_security_regression_count.sh` as a pre-push and CI gate.
