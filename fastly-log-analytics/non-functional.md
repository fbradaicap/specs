---
repo: fastly-log-analytics
spec_type: non_functional
commit: e14824c51e129edb28dd2ccc91ee73e5475323e9
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ec2cfbed4740073039b63156e26fcd0c108f38c3096973c381734f6e64550069
generated_at: 2026-06-30T14:58:31.107203435+02:00
generator: specsync
---

## Performance

- **DuckDB memory limit**: `DUCKDB_MEMORY_LIMIT=4GB` is set as a container environment variable and applied to the DuckDB in-process engine for the backend.
- **Memory allocator**: `libjemalloc2` is preloaded via `LD_PRELOAD=libjemalloc.so.2` in the backend container to reduce heap fragmentation under analytical workloads.
- **Arrow memory pool**: `ARROW_DEFAULT_MEMORY_POOL=system` is configured, routing Arrow allocations through the system (jemalloc) allocator.
- **Connection/pool coalescing**: Directory-size scans use a stale-while-revalidate pattern with a 30-second TTL (`_DIR_SIZE_TTL_S = 30.0`) and per-path cold-locks to collapse concurrent first-arrival walks into a single O(files-in-tree) scan. Background refresh is dispatched to a daemon thread to avoid blocking request threads.
- **CDN fetch timeout**: `urllib.request.urlopen` calls to the CDN are issued with `timeout=30` seconds.
- **Streaming downloads**: File downloads (single file, folder ZIP, full-service ZIP) use `StreamingResponse` with a bounded queue (`_AbortableQueue(maxsize=10)`) backed by daemon threads; chunk size for CDN reads is 65,536 bytes.
- **Fastly Stats API memo cache**: The dominant-cost Fastly Stats API fetch (noted as ~1.8 s p95) is cached server-side with a 45-second TTL (`_FASTLY_COUNTS_TTL = 45.0`); DuckDB `COUNT(*)` results for log-accounting share the same 45-second TTL (`_DUCKDB_COUNTS_TTL = 45.0`). React Query `staleTime` on the frontend is noted as 30 seconds.
- **Bytecode compilation**: Intentionally disabled at image build time (`PYTHONDONTWRITEBYTECODE=1`); cost shifts to container startup (a few seconds on first import) rather than per-request latency.
- **Next.js build cache**: Webpack filesystem and SWC compiled-module caches are mounted at build time (`/app/.next/cache`), keyed by source content hash; stripped from the final image.
- **npm build cache**: npm tarball cache is mounted across builds (`/root/.npm`), keyed by `package-lock.json` SHA-512 integrity hashes.
- **Frontend Node.js requirement**: `node >= 24` (engines field in `package.json`).
- **Wasm scorer**: Compiled with `opt-level = "s"`, `lto = true`, `codegen-units = 1`, `strip = "symbols"`, `panic = "abort"` to minimise binary size and cold-start instantiation time on Fastly Compute (instance-per-request model).
- **Performance gate**: A nightly CI job (`perf-nightly.yml`) runs `perf_gate.sh`, comparing `tests/perf/latest.json` against `baseline.json`; regressions fail the gate.
- **Load test tooling**: `scripts/dev/loadtest_probe.sh` provides a read-path latency probe (serial/concurrent/endpoints) against a local backend. `scripts/loadtest_generator.py` generates synthetic Parquet/rollup data for reproducible perf runs.

## Scalability

- **Deployment topology (local/dev)**: Three containers (backend, frontend, Caddy) share a single network namespace (`network_mode: "service:backend"`). Caddy is the sole ingress on `:80`; frontend runs on `:3000` and backend on `:8000` internally.
- **Production topology**: `docker-compose.prod.yml` overrides to `network_mode: host` (referenced but not included in snapshot). A custom Caddy build with the `caddy-ratelimit` plugin is used in production.
- **Statelessness**: The backend is largely stateless across requests; per-process in-memory caches (dir-size, DuckDB pool stats, Fastly counts) are process-local and tolerate staleness via TTL. The DuckDB file is process-exclusive (noted: `DBBusyError` if a second process attempts access).
- **DuckDB exclusivity constraint**: DuckDB holds an exclusive write lock on the per-service `.duckdb` file; only one backend process can operate against a given service file at a time. This is an explicit horizontal scaling constraint for the backend.
- **Replica counts**: _Not determinable from code._ (No Kubernetes manifests, ECS task definitions, or explicit replica counts are present in the snapshot.)
- **Autoscaling**: _Not determinable from code._
- **Partitioning/sharding**: Data is partitioned per Fastly service (`service_id`) with per-service DuckDB files, Parquet cache directories, and Iceberg table paths. Log-accounting and rollup operations are scoped per service.
- **Rate limiting**: Caddy production image includes the `caddy-ratelimit` plugin (`github.com/mholt/caddy-ratelimit`); specific rate-limit rules are defined in the Caddyfile (not included in snapshot).
- **Frontend**: Next.js 16 with standalone output mode (`server.js`); stateless SSR. Node.js `>=24` required.
- **Edge scorer**: Fastly Compute Wasm runs as an instance-per-request edge function — horizontally scaled by the Fastly network by design.

## Security

- **Transport**: Caddy is the sole externally-facing socket; TLS termination is implied by production Caddy deployment (Caddyfile not included). Custom Caddy binary with `cap_net_bind_service` filecap allows non-root binding on `:80`/`:443`.
- **Privilege separation**:
  - Backend container runs as UID/GID 1000 (`app` user, `/sbin/nologin` shell).
  - Frontend container runs as UID 1001 (`nextjs` user).
  - Caddy container runs as a non-root `caddy` user with `cap_net_bind_service=+ep` applied via `setcap`.
- **Remote access control**: `backend/utils/remote_access.py` (`is_request_remote`) classifies requests as local (loopback peer) or remote; only loopback requests are granted local admin trust. The shared network namespace between Caddy/frontend and the backend enforces this trust boundary. Allowed local hostnames are configurable via `LOCAL_HOSTS` environment variable.
- **CDN secret handling**: The `cdn_secret` bearer token is transmitted as an `x-fastly-key` HTTP header in server-side CDN fetches; it is explicitly never embedded in redirect `Location` URLs to prevent leakage into browser history, address bars, `Referer` headers, or HTTP intermediaries (audit finding 009 referenced).
- **Path traversal protection**: Download endpoints resolve and validate `realpath` of requested keys against the service cache directory using `path_within_dir`; absolute paths and `../` traversals are rejected with HTTP 400.
- **Cross-tenant key guard**: Download endpoints require that requested FOS object keys begin with the requesting service's configured prefix, preventing cross-tenant object access within a shared bucket.
- **Secrets in OpenAPI schema**: Build comment explicitly warns that `openapi.json` (served by Next.js) must not contain embedded credentials, example tokens, or internal hostnames; schema is derived from FastAPI route definitions only.
- **AuthN/AuthZ**: _Not determinable from code._ (No explicit authentication middleware, JWT validation, OAuth, or API key enforcement is visible in the sampled source. Access control appears to be network-topology-based — loopback vs. remote classification — rather than credential-based for admin endpoints.)
- **Input validation**: FastAPI/Pydantic models are used for request body validation. Query parameters use FastAPI `Query()` with constraints (e.g., `ge=0` on `min_files`). Path safety utilities validate filesystem paths.
- **VCL static analysis**: The `falco` VCL linter (pinned to v2.3.0) is installed in the backend container and invoked by `backend.utils.vcl_validator` before publishing custom URL-exclusion regexes to Fastly. `SCORING_REQUIRE_FALCO=1` enables hard-fail mode.
- **Wasm scorer encryption**: Uses `aes-gcm` (pure-Rust, audited) for authenticated encryption in the edge scorer.
- **Session ID entropy**: `getrandom 0.2` provides real entropy for session ID generation in the Wasm scorer (WASI `random_get` on `wasm32-wasip1`).
- **Dependency vulnerability scanning**: `check_osv.py` runs `osv-scanner` in CI, derives severity, and fails on a configurable threshold (`make ci`, `ci.yml`). Dependency overrides in `package.json` pin specific versions of `ip-address`, `postcss`, `hono`, and `js-yaml` for security reasons.
- **Security regression gate**: `check_security_regression_count.sh` enforces a minimum security-regression test count in pre-push hooks and CI.
- **Console OTel guard**: `check_no_console_otel.sh` in CI prevents `OTEL_EXPORTER=console` from being deployed (ADR-08).
- **Accessibility**: `@axe-core/playwright` and `vitest-axe` are used in E2E and unit tests respectively.

## Observability

- **Health endpoint**: `GET /api/health` — used by Docker healthcheck (`curl -f http://localhost:8000/api/health`; interval 30s, timeout 5s, retries 3, start_period 15s).
- **Admin health snapshot**: `GET /admin/health-snapshot` — returns CPU load averages (1m/5m/15m), vCPU count, memory (total/available/used%), data mount and root disk usage, in-flight cron run list, per-service compaction stats, DuckDB connection-pool wait stats, scheduler liveness (age of newest metric snapshot), and config-backup freshness. Optional `probe_fos=1` adds FOS reachability probe.
- **Metrics/tracing (OpenTelemetry)**: `backend/utils/telemetry.py` is referenced throughout (e.g., `record_call`, `record_cdn_call`, `get_tracked_calls`, `process_context_scope`, `start_call_tracking`). OTel `app.thread_wait_ms` histogram is emitted for DuckDB pool wait stats. The `OTEL_EXPORTER=console` configuration is explicitly prohibited in deployed environments (ADR-08, guarded by CI script).
- **Logging**: Python `logging` module is used throughout the backend (`logging.getLogger(__name__)`). `PYTHONUNBUFFERED=1` ensures logs are not buffered in the container.
- **Usage logging**: `backend.utils.usage_logger` (`flush_usage_log`) is called from streaming worker threads; usage-log endpoints are registered on the admin router.
- **Cron progress tracking**: `backend.cron_progress` (`start_progress`, `list_active_runs`) tracks in-flight cron runs with `run_id`, `service_id`, `task`, and `started_at` (ISO-8601). Exposed via the health snapshot.
- **Web Vitals**: `web-vitals` npm package is used for Real User Monitoring (RUM); a JSONL sink is consumed by `scripts/analyze_web_vitals.py`.
- **Scheduler liveness signal**: The age of the newest `metric_snapshots` row is monitored (SRE-06); a large or null age indicates a dead scheduler thread.
- **Performance metrics**: `scripts/baseline_metrics.sh` snapshots architectural metrics; `tests/perf/latest.json` vs `tests/perf/baseline.json` comparison in nightly CI.
- **DuckDB pool wait stats**: Exposed in health snapshot and streamed to OTel histogram.
- **Log accounting**: `GET /admin/log-accounting` compares Fastly Stats API counts against locally-ingested counts; `GET /admin/metric-history/batch` is also available.

## Reliability

- **Graceful shutdown**: Backend container has `stop_grace_period: 75s` (Docker default is 10s). This pairs with a 60-second scheduler drain budget (`_bounded_scheduler_shutdown`) plus 15 seconds of uvicorn/connection-close headroom, preventing partial ingest buffers and orphaned `in_flight` rows on restart.
- **In-flight recovery**: An `_recover_in_flight` routine reconciles orphan `in_flight` rows on boot, serving as a recovery mechanism for ungraceful shutdowns.
- **Cron idempotency**: Backfill and rollup operations are explicitly documented as idempotent — they skip already-built outputs.
- **Async cron deduplication**: `start_or_resume_cron` / `start_cron_run` prevents duplicate concurrent cron runs; a second caller receives the existing `run_id` (or a 503 `cron_busy` in the rebuild-local-view case where deduplication is intentionally not allowed).
- **Stale-while-revalidate caching**: Directory size, Fastly Stats counts, and DuckDB log-accounting counts all use stale-while-revalidate patterns with background refresh threads, so cache expiry never blocks a request thread (after cold start).
- **Client disconnect handling**: `ClientDisconnected` exception propagates out of aborted streaming queues; `_AbortableQueue.aborted` flag prevents writes to disconnected clients.
- **Error swallowing in sync**: `sync_admin_state` swallows all exceptions so a state-export failure never propagates to the primary request.
- **Fire-and-forget admin state export**: Scheduler reload and state export after mutations are best-effort with `try/except Exception: pass`.
- **Restart policy**: All three containers use `restart: unless-stopped`.
- **Config backup freshness**: `scripts/backup_service_configs.sh` writes a freshness marker read by the admin health card (SRE-11, ADR-13) for DR monitoring.
- **Gap-heal cron**: A scheduler-driven gap-heal cron (`compute_log_accounting`) uses `LOG_ACCOUNTING_LOSS_THRESHOLD` and `LOG_ACCOUNTING_MIN_RUN` thresholds to detect sustained log loss and trigger backfill.
- **DuckDB exclusive lock awareness**: The codebase explicitly documents the single-writer constraint and provides an in-process admin endpoint (`/admin/ingest-logs`) rather than a separate process to avoid `DBBusyError`.
- **Retries / circuit breakers**: _Not determinable from code._ (No explicit retry libraries, circuit-breaker middleware, or exponential backoff configurations are visible in the sampled source.)
- **Dependency healthcheck**: Frontend container waits for `backend: condition: service_healthy` before starting.
