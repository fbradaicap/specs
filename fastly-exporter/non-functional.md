---
repo: fastly-exporter
spec_type: non_functional
commit: bebe68cfaf4ceab83b844e46ddd566b2408ba213
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 1ebc9d177a62ad41f42f7a653648848917c9f9eef32d8bda8269bd9a92885943
generated_at: 2026-06-30T14:50:59.294173194+02:00
generator: specsync
---

## Performance

- **API HTTP client timeout:** 15 seconds (default, configurable via `-api-timeout` flag, range 5–60 s).
- **Real-time stats HTTP client timeout:** 45 seconds (default, configurable via `-rt-timeout` flag, range 45–120 s). The rt.fastly.com endpoint uses long-polling by design, which explains the higher default.
- **Polling intervals (configurable):**
  - Service metadata refresh: 1 minute (range 15 s–10 m, `-service-refresh`).
  - Datacenter metadata refresh: 10 minutes (range 10 m–1 h, `-datacenter-refresh`).
  - Product entitlement refresh: 10 minutes (range 10 m–24 h, `-product-refresh`).
  - Dictionary metadata refresh: 5 minutes (range 1 m–24 h, `-dictionary-refresh`).
  - TLS certificate metadata refresh: 6 hours (range 10 m–24 h, `-certificate-refresh`); can be disabled by setting to `0`.
- **Pagination:** The certificate cache fetches up to 200 results per page (`maxCertificatesPageSize = 200`) and follows `rel="next"` links to paginate through all results.
- **In-memory caching:** All API results (services, datacenters, products, certificates, dictionaries) are held in in-process mutex-guarded caches. Prometheus scrapes read from the cache without making upstream API calls, decoupling scrape latency from Fastly API latency.
- **JSON parsing:** Uses `github.com/json-iterator/go` (a high-performance drop-in replacement for `encoding/json`) for API response decoding.
- **Hashing:** Uses `github.com/cespare/xxhash` for service-shard ID hashing.
- **Metrics endpoint:** Exposed on `127.0.0.1:8080/metrics` by default (overridable via `-listen`).
- **pprof:** `net/http/pprof` is imported and registered, making Go runtime profiling endpoints available at `/debug/pprof/` on the same listener.
- **Connection/thread pools:** _Not determinable from code._ No explicit `http.Transport` pool tuning (max idle connections, etc.) is visible; the default Go `http.DefaultTransport` settings apply unless overridden by the `userAgentTransport` wrapper.

---

## Scalability

- **Stateless exporter pattern:** The service holds only soft-state caches rebuilt on each refresh cycle. No persistent storage is used.
- **Horizontal sharding:** Supports a `-service-shard n/m` flag that restricts the instance to only the services whose hashed IDs modulo `m` equal `n-1`. This allows `m` independent exporter instances to partition the full service list, enabling horizontal fan-out.
- **Replica counts / autoscaling:** _Not determinable from code._ No Kubernetes manifests or autoscaling configuration are present in the snapshot (a Helm chart is referenced externally but not included).
- **Concurrency model:** Uses `github.com/oklog/run` for structured goroutine lifecycle management and `golang.org/x/sync/errgroup` for concurrent sub-operations within the main startup sequence.
- **Vertical scaling:** _Not determinable from code._ No CPU/memory resource limits are specified in the included artefacts.

---

## Security

- **Authentication to Fastly API:** All upstream requests carry a `Fastly-Key: <token>` HTTP request header. The token is supplied via the `-token` CLI flag or the `FASTLY_API_TOKEN` environment variable (preferred for secret handling to avoid token exposure in process lists).
- **TLS certificate management permissions:** The API token must hold TLS Management permissions to enable certificate metrics; if a 403 is received on the first certificate refresh, that feature is automatically disabled.
- **Transport security (outbound):** All Fastly API calls target `https://api.fastly.com` and `https://rt.fastly.com` (TLS). The Docker image copies the system CA bundle (`ca-certificates.crt`) from the builder stage into the `scratch` final image to enable TLS verification.
- **Transport security (inbound):** The Prometheus metrics endpoint does not configure TLS; it listens on plain HTTP. Securing the `/metrics` endpoint (e.g., with a reverse proxy or network policy) is the operator's responsibility.
- **User-Agent injection:** A custom `userAgentTransport` wraps all outbound HTTP calls to set a versioned `User-Agent` header, which is standard practice but does not constitute a security control.
- **Principle of least privilege (container):** The Docker image runs as a dedicated non-root user (`fastly-exporter`) in a non-root group, and uses a `scratch` base image to minimise attack surface.
- **Input validation:** Service/metric allow-lists and block-lists are regex-filtered at configuration time. No additional input sanitisation of API responses beyond standard JSON decoding is visible.
- **Secrets handling:** The token may also be set via a plain-text config file (`-config-file`); operators should protect this file with appropriate filesystem permissions.
- **AuthN/AuthZ for `/metrics`:** _Not determinable from code._ No authentication middleware is applied to the Prometheus scrape endpoint.

---

## Observability

- **Metrics:** Exposes Prometheus metrics at `/metrics` (default `127.0.0.1:8080/metrics`) using `github.com/prometheus/client_golang`. Metric families include:
  - `fastly_rt_*` — real-time CDN statistics per service/datacenter.
  - `fastly_rt_datacenter_info` — datacenter metadata gauge.
  - `fastly_rt_cert_expiry_timestamp_seconds` — TLS certificate expiry timestamps.
  - `fastly_rt_dictionary_item_count` / `fastly_rt_dictionary_last_updated_timestamp_seconds` — edge dictionary metadata.
  - Standard Go runtime and process metrics exposed by `prometheus/client_golang`.
- **Prometheus namespace:** Configurable via `-namespace` flag (default: `fastly`); subsystem is fixed to `rt`.
- **Logging:** Structured logfmt logging via `github.com/go-kit/log` to `stderr`. Log level is `info` by default; `debug` level is enabled with the `-debug` flag. Debug log entries include cache refresh durations (e.g., `certificate_refresh_took`) and result counts.
- **Profiling:** `net/http/pprof` handlers are registered on the metrics HTTP listener, enabling on-demand CPU, heap, goroutine, and other Go runtime profiles at `/debug/pprof/`.
- **Health/readiness endpoints:** _Not determinable from code._ No dedicated `/healthz` or `/readyz` endpoint is visible; liveness can be inferred from a successful HTTP response on the metrics port.
- **Tracing:** _Not determinable from code._ No distributed tracing integration (e.g., OpenTelemetry) is present.
- **Version reporting:** The binary embeds version and branch metadata (injected at build time via `-ldflags`) and surfaces them through `prometheus/common/version` (which registers `process_*` and `go_*` info metrics) and the `-version` flag.

---

## Reliability

- **Graceful shutdown:** `github.com/oklog/run` manages all goroutines and ensures clean shutdown on signal receipt or component failure.
- **Polling decoupled from scrape:** Cache refresh goroutines run independently of Prometheus scrapes. A slow or unavailable Fastly API does not block or degrade `/metrics` responses — the last successful cache is served.
- **Graceful degradation on permission errors:** If the certificates API returns HTTP 403 on the first request, the certificate refresh is automatically and permanently disabled rather than returning errors on every subsequent poll.
- **Refresh interval enforcement:** All configurable refresh intervals have hard lower and upper bounds enforced at startup with warning log messages; values outside the valid range are silently clamped.
- **Error propagation:** API errors during polling are logged and, in most cases, cause the refresh cycle to return an error (which will be retried on the next scheduled tick). Per-dictionary fetch errors are logged as warnings and skipped individually, preventing a single dictionary failure from aborting the whole refresh.
- **Retry / circuit breaker:** _Not determinable from code._ No explicit retry logic, exponential back-off, or circuit breaker pattern is implemented; error handling relies on the next scheduled refresh tick.
- **Idempotency:** Cache refreshes are idempotent — each successful refresh fully replaces the prior cached state under a mutex, ensuring consistent reads.
- **Concurrency safety:** All shared cache structures are protected by `sync.Mutex` or `sync.RWMutex` to prevent data races between refresh goroutines and Prometheus scrape callbacks.
- **Docker base image:** Built on `scratch` with a statically linked binary (`CGO_ENABLED=0`, `-a` rebuild flag), eliminating shared library dependencies and reducing the risk of environment-level failures.
