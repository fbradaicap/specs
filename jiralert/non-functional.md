---
repo: jiralert
spec_type: non_functional
commit: 3567c32d289b72a6249df61a4f00fbd1018f62e4
model: claude-sonnet-4-6
prompt_version: v1
input_hash: b0230a420a46e50880b88948a4b8d7f32c217c53ec64df989cb4f39d4315723e
generated_at: 2026-06-30T14:54:46.099632837+02:00
generator: specsync
---

## Performance

- **Default listen address:** `:9097`; no explicit request timeout or read/write deadline is configured on the `http.Server` — the default Go `net/http` server (no timeout) is used. _Not determinable from code_ whether a timeout wrapper is added elsewhere.
- **JIRA description length cap:** Hard-coded default of 32 767 characters (matching the Atlassian server limit), tunable at startup via `-max-description-length`. Prevents oversized payloads from being sent to the JIRA API.
- **HTTP connection pooling to JIRA:** Each incoming `/alert` request instantiates a new `jira.Client` (and therefore a new `http.Transport`/connection pool) per request. No connection reuse across requests is implemented; a TODO comment in the source acknowledges this gap. Effective concurrency is therefore limited by the number of simultaneous in-flight webhook requests and the OS TCP stack.
- **Go runtime block/mutex profiling:** Enabled at rate 1 when the `DEBUG` environment variable is set; disabled otherwise to avoid profiling overhead in production.
- **pprof:** Registered via `_ "net/http/pprof"` blank import; pprof endpoints are available on the same port for ad-hoc CPU/memory profiling.
- Caching: _Not determinable from code._
- Thread/goroutine pool configuration: _Not determinable from code._

## Scalability

- **Stateless design:** No in-process state is shared between requests beyond the loaded configuration and template objects (read-only after startup). The service can be horizontally scaled behind a load balancer. (target)
- **Replica count / autoscaling:** _Not determinable from code._ No Kubernetes manifests, HPA, or replica settings are present in the snapshot.
- **Configuration hot-reload:** A `/config` endpoint exposes the current configuration; reloading requires a process restart (no SIGHUP handler is visible).
- **Partitioning/sharding:** _Not determinable from code._
- The Dockerfile produces a single statically linked binary with no external volume mounts, which is compatible with container-orchestration horizontal scaling.

## Security

- **Authentication to JIRA — Basic Auth:** Username + password supplied via receiver configuration (`conf.User` / `conf.Password`); transported over the JIRA API URL (expected to be HTTPS by operators). Uses `go-jira`'s `BasicAuthTransport`.
- **Authentication to JIRA — Personal Access Token (PAT):** Alternative to Basic Auth; token stored in receiver configuration (`conf.PersonalAccessToken`). Uses `go-jira`'s `PATAuthTransport`. JWT parsing is available transitively via `github.com/golang-jwt/jwt/v4` (pulled in by `go-jira`).
- **Secrets handling:** Passwords and PATs are declared as typed secrets in the YAML config. The README notes support for environment variable substitution in the config file, providing a mechanism to avoid hard-coding secrets on disk.
- **Transport security (inbound):** No TLS configuration is present for the HTTP listener; TLS termination is expected to be handled externally (e.g., reverse proxy or ingress). (target)
- **Transport security (outbound):** Depends on the scheme of `conf.APIURL`; no explicit TLS enforcement or certificate pinning is coded.
- **AuthN on webhook endpoint:** None — the `/alert` endpoint accepts unauthenticated POST requests. Network-level controls (firewall, ingress policy) are assumed to restrict callers to Alertmanager only.
- **Input validation:** Incoming webhook body is decoded with `json.NewDecoder`; malformed JSON returns HTTP 400. Unknown receiver names return HTTP 404. No additional schema validation beyond JSON decoding is present.
- **Label length safety:** The `-hash-jira-label` flag hashes alert label key-value pairs to avoid exceeding JIRA's 255-character label limit, mitigating a potential injection/overflow vector.

## Observability

- **Metrics:** `github.com/prometheus/client_golang` is used; a `promhttp.Handler()` is mounted at `/metrics`, exposing standard Go runtime metrics and a `requestTotal` counter labelled by receiver name and HTTP status code.
- **Logging:** Structured logging via `github.com/go-kit/log` with configurable level (`-log.level`: debug/info/warn/error) and format (`-log.format`: logfmt or JSON). Startup, configuration load errors, template errors, and per-request outcomes are logged.
- **Health endpoint:** `GET /healthz` returns HTTP 200 `"OK"` unconditionally; suitable for liveness probes.
- **Config endpoint:** `GET /config` exposes the parsed configuration (receiver names and settings) for operational inspection.
- **Home/index endpoint:** `GET /` serves an HTML index page (via `HomeHandlerFunc`).
- **Distributed tracing:** _Not determinable from code._ No tracing library or trace-context propagation is present.
- **Readiness probe:** _Not determinable from code._ `/healthz` can be used but no distinct readiness endpoint is implemented.
- **pprof:** Available at `/debug/pprof/*` via the blank `net/http/pprof` import; useful for runtime profiling in production without a restart.

## Reliability

- **Retry signalling:** The `notify.Receiver.Notify()` method returns a boolean `retry` flag. When `true`, the handler responds with HTTP 503 (Service Unavailable), instructing Alertmanager to retry the webhook delivery according to its own retry schedule. When `false`, HTTP 400 is returned to suppress retries.
- **Idempotency:** One JIRA issue is created per distinct Alertmanager group key. Duplicate calls for the same group key will find the existing issue and update or reopen it rather than creating duplicates, making the handler effectively idempotent.
- **Resolved-alert handling:** Resolved alerts do not close JIRA issues by default (human action expected). Optionally, `auto_resolve` configuration can transition issues to a target state. Issues with a `wont_fix_resolution` are intentionally skipped to avoid undesired reopening.
- **Reopen guard:** The `-reopen-tickets=false` flag can disable ticket reopening entirely, providing a safety valve.
- **Circuit breakers / bulkheads:** _Not determinable from code._ No circuit-breaker library is present; failures are propagated directly to Alertmanager via the retry/no-retry response codes.
- **Timeout on outbound JIRA calls:** _Not determinable from code._ No explicit `http.Client` timeout is set on the transports created per request; a slow JIRA instance could cause goroutine accumulation.
- **Graceful shutdown:** _Not determinable from code._ No `http.Server.Shutdown` or signal handler is visible in the sampled source.
- **Failure isolation:** Because a new JIRA client is created per request, a failure for one receiver does not affect in-flight requests for other receivers.
