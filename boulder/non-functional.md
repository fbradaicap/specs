---
repo: boulder
spec_type: non_functional
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:58:58.292168945+02:00
generator: specsync
---

## Performance

- **Transport protocol**: All internal service communication uses gRPC over Protocol Buffers, providing compact binary serialisation and HTTP/2 multiplexing across all RPC endpoints (RA, CA, SA, VA, Publisher, CRL, OCSP, Nonce, Akamai Purger).
- **Streaming RPCs**: Several SA operations (`GetRevokedCerts`, `GetSerialsByAccount`, `GetSerialsByKey`, `SerialsForIncident`, `GenerateCRL`, `UploadCRL`) use gRPC server/client streaming to avoid buffering large result sets in memory, reducing latency for bulk read paths.
- **Database**: MySQL (MariaDB 10.5) is fronted by ProxySQL 2.5.4 (`boulder-proxysql:6033`), which provides connection pooling and query routing. Slow-query logging is enabled on the MariaDB instance (`--slow-query-log --log-output=TABLE`). The `go-sql-driver/mysql` is pinned to v1.5.0 explicitly because later versions introduced performance regressions (documented in `go.mod`).
- **Caching**: `github.com/golang/groupcache` is a direct dependency, indicating in-process or distributed caching for read-heavy paths (e.g., certificate status, OCSP responses). Redis (v6.2.7) is deployed in two distinct clusters: one for OCSP response caching (`redis-ocsp.config`, nodes at 10.33.33.2–3) and one for rate-limit counters (`redis-ratelimits.config`, nodes at 10.33.33.4–5), accessed via `github.com/redis/go-redis/v9`.
- **Partitioned tables**: High-volume tables (`authz2`, `certificateStatus`, `certificates`, `fqdnSets`, `issuedNames`, `orderFqdnSets`, `orderToAuthz2`) are `PARTITION BY RANGE(id)`, enabling partition pruning and incremental archival.
- **Timeouts**: Specific timeout values are not determinable from the provided files; `cenkalti/backoff/v4` is present for retry/backoff policies but numeric values require configuration file inspection. _Not determinable from code._
- **Connection pool sizes**: _Not determinable from code._

## Scalability

- **Horizontal scaling**: Boulder is architecturally decomposed into independently deployable components (RA, CA, SA, VA, Publisher, CRLStorer, OCSP Generator, Nonce Service, Akamai Purger), each addressable as a separate gRPC service. This allows each component to be scaled independently.
- **Statelessness**: The RA, CA, VA, Publisher, and Nonce services are stateless with respect to persistent data (all state is held in MySQL or Redis), enabling horizontal replication. The Nonce service's `Nonce`/`Redeem` RPC pair implies nonce state is externalised (likely to Redis).
- **Service discovery**: Consul (hashicorp/consul 1.15.4) is used for service discovery, with gRPC clients configured via `ServerAddress` fields resolvable via Consul A records (e.g., `ra.service.consul`). SRV-record-based discovery is indicated as a future migration target in `docker-compose.yml`.
- **Read/write split**: The SA exposes a `StorageAuthorityReadOnly` service alongside the full `StorageAuthority`, allowing read replicas to serve read-only workloads and reduce write-master load.
- **Replica counts**: _Not determinable from code._ (No Kubernetes or Compose replica fields are specified in the provided manifests.)
- **Autoscaling**: _Not determinable from code._
- **Database partitioning**: Range partitioning on large tables (see Performance) supports horizontal shard-like growth and archival; `crlShards` table provides explicit CRL sharding by issuer and shard index.

## Security

- **Transport security**: All gRPC communication is expected to use mutual TLS; `grpc-ecosystem/go-grpc-prometheus` and `otelgrpc` interceptors wrap gRPC servers and clients. Specific mTLS configuration values are not present in the provided files but are standard for Boulder's architecture.
- **HSM / PKCS#11**: `github.com/letsencrypt/pkcs11key/v4` and `github.com/miekg/pkcs11` are direct dependencies, indicating CA private keys are stored in and operations performed by a hardware (or software, e.g., SoftHSM) security module. The `docker-compose.yml` mounts `.softhsm-tokens/` into the container.
- **Key blocking**: A `blockedKeys` table and corresponding `AddBlockedKey`/`KeyBlocked` RPCs implement a key denylist to prevent reuse of compromised or known-weak keys. `github.com/titanous/rocacheck` provides ROCA (weak RSA key) detection.
- **ACME protocol security**: JWS/JWK handling via `gopkg.in/go-jose/go-jose.v2`; nonce anti-replay enforced through the dedicated `NonceService` (`Nonce`/`Redeem` RPCs).
- **Input validation**: `github.com/letsencrypt/validator/v10` is used for struct-level input validation. Certificate linting uses `github.com/zmap/zlint/v3` and `github.com/zmap/zcrypto` pre-issuance. Public suffix validation uses `github.com/weppos/publicsuffix-go`.
- **CAA checking**: Dedicated `CAA.IsCAAValid` RPC in the VA enforces DNS CAA policy before issuance.
- **Secrets handling**: _Not determinable from code._ No secrets manager integration is visible in the provided snapshot beyond PKCS#11 for key material.
- **Rate limiting**: Redis-backed rate-limit counters (`ratelimits/` package, dedicated Redis cluster) enforce issuance rate limits per registration, IP, and FQDN set.
- **AuthN/AuthZ between services**: Service-to-service authorisation relies on gRPC with (expected) mTLS client certificates; no application-layer bearer token mechanism is visible in the provided proto definitions.

## Observability

- **Metrics**: `github.com/prometheus/client_golang` (v1.15.1) and `github.com/grpc-ecosystem/go-grpc-prometheus` provide Prometheus metrics for all gRPC servers and clients. `github.com/prometheus/client_model` is also present. HTTP metrics are covered by `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp`.
- **Distributed tracing**: OpenTelemetry tracing is fully integrated: `go.opentelemetry.io/otel`, `otel/sdk`, `otel/trace`, `otelgrpc` (gRPC interceptors), `otelhttp` (HTTP middleware), and `otlptracegrpc` (OTLP exporter over gRPC) are all direct dependencies. Jaeger (`jaegertracing/all-in-one:1.50` at 10.77.77.17) is the trace backend in the development/CI environment.
- **Logging**: `github.com/nxadm/tail` is present for log file tailing. The `log/` package exists in the source tree. rsyslog is configured inside the Docker image (`boulder.rsyslog.conf`), indicating structured syslog-based log forwarding. `go-logr/logr` and `go-logr/stdr` provide a standard logging interface.
- **Health/readiness endpoints**: _Not determinable from code._ No explicit `/healthz` or `/readyz` HTTP handler registrations are visible in the provided files.
- **Observer component**: A dedicated `observer/` package exists, suggesting an active monitoring or probing subsystem within the service.
- **Slow query monitoring**: MariaDB is configured with `--slow-query-log --log-output=TABLE`, making slow queries queryable from the `mysql.slow_log` table during integration tests.

## Reliability

- **Retry and backoff**: `github.com/cenkalti/backoff/v4` is a direct dependency, providing exponential-backoff retry logic for transient failures (exact usage sites are not determinable from the provided snapshot).
- **Idempotency**: The SA's write RPCs (`AddCertificate`, `AddPrecertificate`, `AddSerial`, `FinalizeAuthorization2`, etc.) follow an append/finalise pattern consistent with idempotent operations. The `SetOrderProcessing`/`SetOrderError` state machine prevents double-issuance.
- **CRL shard leasing**: `LeaseCRLShard`/`UpdateCRLShard` RPCs with `leasedUntil` column in `crlShards` provide distributed mutual exclusion for CRL generation workers, preventing split-brain CRL updates.
- **Read/write separation**: `StorageAuthorityReadOnly` allows the system to continue serving read traffic even when write-path availability is degraded.
- **ProxySQL**: Database connection proxying via ProxySQL 2.5.4 provides connection pooling, failover, and query routing, improving SA resilience against database node failures.
- **Redis clustering**: Two separate Redis clusters (OCSP and rate limits) provide fault isolation between the OCSP caching path and rate-limiting path; `github.com/dgryski/go-rendezvous` (consistent hashing) is present for Redis cluster key distribution.
- **Circuit breakers**: _Not determinable from code._ No circuit-breaker library (e.g., `hystrix`, `gobreaker`) is present in the dependency list.
- **gRPC deadline propagation**: Standard gRPC deadline propagation is available via the Go gRPC library; specific per-RPC deadline values are _not determinable from code._
- **Precertificate/certificate hash binding**: `certProfileHash` is passed between RA and CA through the issuance flow to detect profile mutation across round-trips, providing a consistency check that prevents silent mis-issuance under failure conditions.
