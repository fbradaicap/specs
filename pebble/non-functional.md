---
repo: pebble
spec_type: non_functional
commit: b37975ee86323dde963d5087b3103d2af34484fd
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9c647701705747ae93296249d39425b915371b7b8aaf4a2a3ebca6fb5d93a790
generated_at: 2026-06-30T14:54:09.090550209+02:00
generator: specsync
---

## Performance

- The ACME API is served over HTTPS on port 14000 and the Management API on port 15000 (as configured in `docker-compose.yml`). No explicit latency or throughput targets are defined in code or configuration.
- No HTTP server `ReadTimeout`, `WriteTimeout`, or `IdleTimeout` values are set on the `http.Server` instances created in `cmd/pebble/main.go` or `cmd/pebble-challtestsrv/main.go`; Go's default (no timeout) applies.
- The management server in `pebble-challtestsrv` likewise has no timeout configured on its `http.Server` struct.
- No connection pool sizing, worker pool, or goroutine-pool configuration is present; Go's standard `net/http` per-connection goroutine model is used implicitly.
- All certificate/order/authorization state is held in an in-process `db.NewMemoryStore()` (no external DB); lookup and write latency is bounded only by in-memory operations and Go's garbage collector.
- No caching layer is configured.
- `RetryAfter.Authz` and `RetryAfter.Order` are configurable integer values (seconds) read from the JSON config file, controlling `Retry-After` headers returned to ACME clients. Actual values are deployment-specific and not fixed in code.
- `CertificateValidityPeriod` is a configurable `uint64` field; no default is enforced in code beyond zero-value behaviour.

## Scalability

- The service is designed as a single-process, in-memory ACME test server. All state (`db.NewMemoryStore()`) is local to the process, making horizontal scaling with shared state **not supported** without code changes.
- The `docker-compose.yml` defines exactly one replica each for `pebble` and `challtestsrv`; no autoscaling or replica-count configuration is present.
- The `Dockerfile.release` is multi-platform (Linux/Windows, multi-arch via `TARGETPLATFORM`/`TARGETARCH` build arguments), supporting vertical deployment across architectures, but no horizontal scaling mechanism is configured.
- No partitioning, sharding, or distributed coordination is implemented.
- Statelessness: the service is **stateful** within a single process lifetime; restarting the process discards all issued certificates and accounts.

## Security

- **Transport security:** All ACME and Management API traffic is served exclusively over TLS (`http.ListenAndServeTLS`), using a certificate and private key path supplied via the JSON config (`Certificate`, `PrivateKey` fields). The challtestsrv management interface uses plain HTTP (no TLS) on port 8055.
- **AuthN/AuthZ:** ACME request authentication relies on JWS (JSON Web Signatures) via `github.com/go-jose/go-jose/v4` and `golang.org/x/crypto`. External Account Binding (EAB) can be enforced via the `ExternalAccountBindingRequired` flag and pre-shared MAC keys (`ExternalAccountMACKeys` map in config).
- **Domain blocklist:** A configurable `DomainBlocklist` is loaded at startup and stored in the memory DB, blocking certificate issuance for specified domains.
- **Secrets handling:** EAB MAC keys are read from the JSON configuration file at startup. No secrets-management integration (e.g., Vault, environment-variable injection beyond `PEBBLE_ALTERNATE_ROOTS`/`PEBBLE_CHAIN_LENGTH`) is present in the code.
- **Input validation:** JWS-signed ACME requests are validated by the WFE layer; strict mode (`-strict` flag) enables additional API conformance checks.
- **Network isolation:** In the Docker Compose deployment, pebble and challtestsrv communicate over an isolated bridge network (`acmenet`, `10.30.50.0/24`).
- **DoH:** The challtestsrv optionally serves DNS-over-HTTPS with a configurable certificate/key pair.
- _No rate limiting, IP allowlisting, or authentication on the challtestsrv management HTTP interface is evident in the code._

## Observability

- **Logging:** Both `pebble` and `pebble-challtestsrv` use Go's standard `log.Logger` writing to stdout (`log.New(os.Stdout, …, log.LstdFlags)`). Log entries are prefixed with `"Pebble "` and `"pebble-challtestsrv - "` respectively.
- **Startup logging:** The main process logs the listen address, ACME directory URL, and Management API URL (including root CA certificate paths) at startup.
- **Metrics:** _Not determinable from code._ No Prometheus, OpenTelemetry, or other metrics instrumentation is present.
- **Distributed tracing:** _Not determinable from code._ No tracing library or instrumentation is present.
- **Health/readiness endpoints:** _Not determinable from code._ No dedicated `/healthz`, `/readyz`, or equivalent endpoint is registered in the source samples or configuration.

## Reliability

- **Availability design intent:** Pebble is explicitly a test/development ACME server (not a production CA); no high-availability, replication, or disaster-recovery provisions are present.
- **Retries:** No client-side retry logic or retry middleware is implemented in the server. `RetryAfter` values for authorizations and orders are configurable, instructing ACME clients how long to wait before polling.
- **Circuit breakers:** _Not determinable from code._ No circuit-breaker library or pattern is used.
- **Timeouts:** No explicit HTTP server-side read/write/idle timeouts are configured (see Performance). No outbound HTTP client timeout is evident in the source samples for VA challenge validation.
- **Idempotency:** ACME protocol semantics (nonce-based replay protection via JWS) provide request-level idempotency controls; this is handled by the `go-jose` library.
- **Failure handling:** `cmd.FailOnError` is used throughout startup for unrecoverable configuration errors, causing the process to exit immediately. Individual goroutine errors (e.g., management server shutdown) are logged but do not propagate.
- **State durability:** All state is in-memory only; a process crash or restart results in complete loss of all ACME accounts, orders, and issued certificates. No persistence or backup mechanism is present.
- **Signal handling:** `cmd.CatchSignals` is used in `pebble-challtestsrv` to perform graceful shutdown of sub-servers on OS signal receipt.
