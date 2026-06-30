---
repo: fastly-log-analytics
spec_type: non_functional
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:57:16.978197709+02:00
generator: specsync
---

## Performance

- **DuckDB memory limit**: configured at `DUCKDB_MEMORY_LIMIT=4GB` via environment variable in both `docker-compose.yml` and the backend `Dockerfile`.
- **Memory allocator**: `libjemalloc2` is preloaded (`LD_PRELOAD=libjemalloc.so.2`) in the backend container to improve heap allocation throughput and fragmentation under DuckDB workloads.
- **Arrow memory pool**: `ARROW_DEFAULT_MEMORY_POOL=system` is set, routing Arrow allocations through the system (jemalloc) allocator.
- **Directory-size cache**: Per-path TTL of 30 seconds (`_DIR_SIZE_TTL_S = 30.0`) with stale-while-revalidate semantics and coalesced background refresh threads to avoid O(files-in-tree) walks on every request.
- **Log-accounting cache**: Fastly Stats API results cached with a 45-second TTL (`_FASTLY_COUNTS_TTL = 45.0`); DuckDB COUNT results cached with a matching 45-second TTL (`_DUCKDB_COUNTS_TTL = 45.0`). React Query `staleTime` is noted as 30 seconds on the client side.
- **CDN file fetch timeout**: `urllib.request.urlopen(req, timeout=30)` — 30-second timeout applied to CDN/FOS file fetches in download endpoints.
- **Streaming responses**: ZIP downloads are produced via a daemon-thread producer/queue consumer pattern (`_QueueFile`, `_AbortableQueue`, `_stream_from_worker`) with a bounded queue (`maxsize=10`) to avoid unbounded buffering.
- **Bytecode compilation**: Intentionally disabled at image build time (`PYTHONDONTWRITEBYTECODE=1`); cost shifts to container startup (a few seconds of import overhead) rather than per-request latency.
- **WASM scorer binary**: Compiled with `opt-level = "s"`, `lto = true`, `codegen-units = 1`, `strip = "symbols"`, and `panic = "abort"` to minimise binary size and cold-start instantiation time on the Fastly Compute edge (instance-per-request model).
- **Shutdown drain**: Backend container `stop_grace_period: 75s` — paired with a 60-second scheduler drain budget plus 15 seconds of uvicorn/connection-close headroom to avoid mid-chunk ingest truncation on restart.
- **Frontend**: Next.js 16 with standalone output; `NEXT_TELEMETRY_DISABLED=1`; webpack and SWC caches retained across builds via BuildKit cache mounts. Node ≥ 24 required.
- **Performance gate**: A nightly `perf-nightly.yml` CI workflow compares `tests/perf/latest.json` against `baseline.json` via `scripts/perf_gate.sh`; regressions block merge (target).

## Scalability

- **Single-instance, single-host deployment**: `docker-compose.yml` and `docker-compose.prod.yml` describe a single-VM topology. The backend, frontend, and Caddy share one network namespace (`network_mode: "service:backend"`). No replica count, autoscaling, or orchestrator (Kubernetes/ECS) configuration is evident.
- **DuckDB exclusive lock constraint**: Source comments explicitly document that DuckDB holds an exclusive lock on the per-service `.duckdb` file; external processes cannot open the database concurrently even read-only. This is a hard vertical-scaling constraint — horizontal scaling of the backend process is not supported without a DuckDB replacement or sharding by service.
- **Stateful local cache**: The backend reads and writes a local `/app/cache` and `/app/data` volume. State is not replicated or shared, precluding stateless horizontal scaling of the backend tier.
- **Edge scorer (Fastly Compute)**: The `session-scorer` Wasm runs on Fastly's edge network in an instance-per-request model, inheriting Fastly's CDN-level horizontal scaling. The scoring matrix is distributed via the Fastly KV Store.
- **Frontend (Next.js standalone)**: Stateless SSR server; horizontally scalable in principle, but no load balancer or replica configuration is present in the repository.
- **Partitioning**: Log data is partitioned by service ID and by hour/day in the local Parquet/Iceberg layout; this is a data-organisation strategy, not a distributed-sharding mechanism.

_Autoscaling configuration is not determinable from code._

## Security

- **Network trust model**: The backend distinguishes loopback (local admin, trusted) from remote peers via `backend/utils/remote_access.py:is_request_remote`. The shared network namespace (`network_mode: "service:backend"`) ensures Caddy and the Next.js SSR tier reach the backend over `127.0.0.1`, qualifying as local-admin peers. Remote requests receive restricted access.
- **Allowed hosts**: Configurable via `LOCAL_HOSTS` environment variable (e.g. `fastly.localhost,fastly.analytics`); `localhost`/`127.0.0.1` are permitted by default.
- **CDN secret handling**: The `cdn_secret` field is a shared bearer token. It is transmitted server-side as an `x-fastly-key` HTTP header rather than embedded in redirect `Location` URLs, explicitly to prevent leakage into browser history, the `Referer` header, and HTTP intermediaries (referenced as audit finding 009).
- **Cross-tenant path guard**: Download endpoints enforce that requested object keys begin with the service's configured prefix before minting presigned URLs or CDN redirects, preventing cross-tenant FOS object access.
- **Path traversal prevention**: `backend/utils/path_safety:path_within_dir` resolves both the cache directory and the candidate path with `os.path.realpath` and asserts `commonpath == cache_dir`, rejecting absolute paths and `../`-traversal payloads.
- **Non-root containers**: Both the backend (`uid/gid 1000`, user `app`) and Caddy (`uid/gid` recreated via `addgroup/adduser`, user `caddy`) run as non-root users. The frontend runs as `uid 1001` (`nextjs`).
- **Caddy rate limiting**: The production Caddy image is a custom build that includes the `caddy-ratelimit` plugin (`mholt/caddy-ratelimit`), providing ingress-level rate limiting.
- **Transport security**: Caddy is the sole external-facing ingress. The production compose uses `network_mode: host`; TLS termination and HTTPS enforcement are expected to be handled by Caddy but the specific `Caddyfile.prod` is not present in the snapshot. _Full TLS configuration is not determinable from the available code._
- **VCL validation**: The `falco` static analyser (pinned to v2.3.0) is installed in the backend image and used to lint VCL scoring snippets before publishing. When `SCORING_REQUIRE_FALCO=1` is set, a missing binary causes a hard failure.
- **OpenAPI schema hygiene**: Build comments note that `openapi.json` must be kept free of embedded credentials, example tokens, or internal hostnames; the schema derives only from FastAPI route definitions.
- **Dependency vulnerability scanning**: `scripts/check_osv.py` runs `osv-scanner` in CI (`ci.yml`, `make ci`) and fails on a severity threshold. A `check_security_regression_count.sh` script enforces a floor on security-regression test count as a pre-push hook and CI gate.
- **OTEL exporter guard**: `scripts/check_no_console_otel.sh` (ADR-08) prevents `OTEL_EXPORTER=console` from being set in deployed environments, guarding against accidental plaintext telemetry emission.
- **AuthN/AuthZ**: _Not determinable from code._ No authentication middleware (e.g. OAuth2, API keys, JWT validation) is visible in the sampled router or middleware code beyond the loopback/remote trust distinction.

## Observability

- **Health check**: `GET /api/health` — polled by Docker's `healthcheck` directive every 30 seconds, with a 5-second timeout and 3 retries; container start grace period is 15 seconds.
- **Admin health snapshot**: `GET /admin/health-snapshot` — returns CPU load averages (1m/5m/15m), vCPU count, memory usage (from `/proc/meminfo`), disk usage of `/app/data` and `/`, in-flight cron run list, DuckDB connection-pool wait statistics, scheduler liveness (age of newest `metric_snapshots` row), per-service compaction stats, and config-backup freshness. Optional `probe_fos=1` parameter triggers a live FOS reachability probe.
- **OpenTelemetry**: `scripts/check_no_console_otel.sh` confirms OTel is configured in deployed environments. DuckDB pool wait samples are streamed to an OTel `app.thread_wait_ms` histogram. `backend/utils/telemetry` provides `record_call`, `record_cdn_call`, `get_tracked_calls`, `process_context_scope`, and `start_call_tracking` utilities; CDN call latency and byte counts are recorded per-request. _Specific OTel exporter endpoint/collector configuration is not determinable from the available code._
- **Logging**: Python standard-library `logging` is used throughout the backend. `PYTHONUNBUFFERED=1` ensures log output is not buffered. Structured logging details are not determinable from the sampled code.
- **Web Vitals / RUM**: The frontend depends on `web-vitals` ^5.3.0; `scripts/analyze_web_vitals.py` processes a JSONL RUM sink.
- **Usage logging**: `backend/utils/usage_logger:flush_usage_log` is called from streaming worker threads; a `record_call` telemetry helper tracks per-request operations. An admin usage-log router registers `/api/admin/usage-log*` and `/api/admin/system-jobs` endpoints.
- **Cron progress**: `backend/cron_progress` provides `start_progress` and `list_active_runs`; in-flight run metadata is exposed via the health-snapshot endpoint.
- **Scheduler liveness signal (SRE-06)**: Age of the newest `metric_snapshots` row, surfaced via the health-snapshot endpoint.
- **Performance baseline metrics**: `scripts/baseline_metrics.sh` snapshots architectural metrics (output gitignored); compared nightly by `scripts/perf_gate.sh`.
- **Accessibility testing**: `@axe-core/playwright` and `vitest-axe` are included in frontend dev dependencies for automated accessibility checks in E2E and unit test suites.

## Reliability

- **Graceful shutdown / drain**: `stop_grace_period: 75s` on the backend container. The backend's `_bounded_scheduler_shutdown` in `main.py` has a 60-second scheduler drain budget. This prevents partial ingest buffers and orphaned `in_flight` rows that would otherwise require reconciliation on the next boot via `_recover_in_flight`.
- **Stale-while-revalidate caching**: The directory-size cache returns stale values immediately on TTL expiry while triggering a single coalesced background refresh thread, preventing cache-stampede saturation of CPU/disk under concurrent `/api/bootstrap` calls (documented as a 2026-06-23 production issue).
- **Concurrency coalescing**: Per-path cold locks (`_dir_size_cold_locks`) ensure that N concurrent first-arrivals for a new cache path produce a single directory walk rather than N concurrent walks.
- **Error isolation in fire-and-forget sync**: `sync_admin_state` in `_state_sync.py` wraps all exceptions and swallows them, ensuring that admin state export or scheduler reload failures never propagate to the primary request.
- **Idempotent backfill operations**: Rollup backfill and compaction endpoints are documented as idempotent — they skip already-built outputs. The `POST /admin/backfill-bundle-rollups` endpoint is explicitly marked idempotent.
- **Asynchronous cron dispatch**: Long-running operations (`/admin/commit-iceberg`, `/admin/rebuild-local-view`, `/admin/ingest-logs`) return `202 Accepted` with a `run_id` for polling, rather than blocking the HTTP response. A `start_or_resume_cron` utility coalesces duplicate trigger requests to an already-running cron instead of spawning duplicates.
- **Cron busy guard (rebuild)**: The rebuild-local-view endpoint explicitly returns `503 cron_busy` rather than piggy-backing on an in-flight run, since the operation involves cache-clearing side effects that must not be silently skipped.
- **Client-disconnect handling**: `_AbortableQueue` sets an `aborted` flag when the streaming generator's `finally` block runs; subsequent `put` calls raise `ClientDisconnected` rather than blocking indefinitely, allowing the ZIP worker thread to terminate cleanly.
- **Fallback scoring matrix**: The backend includes a `matrix.default.json` fallback; `_load_matrix()` returns a meaningful "no signal" response rather than erroring when no trained matrix is present.
- **FOS CDN fallback**: `_fetch_file_to_zip` attempts CDN fetch first and falls back to direct FOS client access on failure.
- **Config backup freshness monitoring (SRE-11/ADR-13)**: `scripts/backup_service_configs.sh` writes a freshness marker read by the admin health card, enabling detection of stale DR backups.
- **Contract testing**: `scripts/run_contract_backend.py` is used as a setup step in both frontend vitest and Playwright E2E test suites, providing backend-contract regression coverage.
- **Retry/circuit-breaker patterns**: _Not determinable from code._ No explicit retry libraries, circuit-breaker middleware, or exponential backoff configuration is visible in the sampled source.
