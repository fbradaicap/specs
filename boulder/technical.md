---
repo: boulder
spec_type: technical
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:57:58.750743927+02:00
generator: specsync
---

## Tech Stack

- **Language:** Go 1.21 (primary); Python (test tooling only)
- **Runtime:** Go 1.21.8 (pinned in `docker-compose.yml` build arg `GO_VERSION`)
- **RPC framework:** gRPC (`google.golang.org/grpc v1.60.1`) with Protocol Buffers (`google.golang.org/protobuf v1.33.0`)
- **Database ORM/driver:** `github.com/letsencrypt/borp` (fork of go-borp ORM); `github.com/go-sql-driver/mysql v1.5.0` (pinned, newer versions excluded due to performance regressions)
- **Caching/rate-limiting store:** `github.com/redis/go-redis/v9 v9.3.0`
- **DNS:** `github.com/miekg/dns v1.1.58`
- **PKCS#11 / HSM:** `github.com/letsencrypt/pkcs11key/v4`, `github.com/miekg/pkcs11 v1.1.1`
- **Cryptography / linting:** `github.com/zmap/zcrypto`, `github.com/zmap/zlint/v3`, `golang.org/x/crypto`, `github.com/titanous/rocacheck`
- **Certificate Transparency:** `github.com/google/certificate-transparency-go v1.1.6`
- **Object storage (CRL upload):** `github.com/aws/aws-sdk-go-v2/service/s3 v1.50.2`
- **Observability:** OpenTelemetry (`go.opentelemetry.io/otel v1.24.0`, OTLP gRPC exporter, otelgrpc + otelhttp instrumentation); Prometheus (`github.com/prometheus/client_golang v1.15.1`)
- **JOSE / ACME:** `gopkg.in/go-jose/go-jose.v2 v2.6.1`; `github.com/eggsampler/acme/v3 v3.5.0` (test client)
- **Public suffix / domain validation:** `github.com/weppos/publicsuffix-go`
- **Groupcache:** `github.com/golang/groupcache`
- **Config:** `gopkg.in/yaml.v3`, `github.com/pelletier/go-toml`
- **Python test deps:** `acme`, `cryptography`, `PyOpenSSL`, `requests`, `matplotlib`, `numpy`, `pandas`
- **Build tooling:** `sql-migrate`, `protoc-gen-go`, `protoc-gen-go-grpc`, `golangci-lint`, `staticcheck`

---

## Architecture Patterns

Boulder is a **multi-component, internally service-oriented** ACME CA backend, structured as a collection of discrete gRPC microservices that communicate over an internal gRPC mesh. The overall style is **layered + service-mesh (internal RPC bus)**, with clear separation of authorities:

| Component | Role |
|---|---|
| **WFE2** (Web Front End) | HTTP/ACME protocol termination, exposes ACMEv2 API on port 4001 |
| **RA** (Registration Authority) | Orchestration: registration, order lifecycle, validation coordination, revocation |
| **CA** (Certificate Authority) | Certificate and precertificate issuance, OCSP generation, CRL generation |
| **VA** (Validation Authority) | Domain control validation (HTTP-01, TLS-ALPN-01, DNS-01); CAA checking |
| **SA** (Storage Authority) | Database abstraction layer; exposes a read-only and a read-write gRPC service over MariaDB |
| **Publisher** | Certificate Transparency log submission |
| **Nonce Service** | ACME replay-nonce issuance and redemption |
| **OCSP Responder** (port 4002/4003) | Serves OCSP responses; backed by Redis cache |
| **CRL Storer** | Uploads signed CRLs to S3-compatible object storage |
| **Akamai Purger** | CDN cache purge for OCSP/CRL URLs |
| **Observer** | Metrics/probe observer component |
| **Rate Limits** | Redis-backed rate limiting subsystem (`ratelimits/`) |

Key patterns:
- **CQRS-adjacent separation** in the SA: `StorageAuthorityReadOnly` and `StorageAuthority` (read-write) are distinct gRPC services.
- **Streaming RPCs** for bulk operations (`GetRevokedCerts`, `GetSerialsByAccount`, `GetSerialsByKey`, `SerialsForIncident`, `GenerateCRL`, `UploadCRL`).
- **Hexagonal / ports-and-adapters** within each component: gRPC handler → core business logic → SA/CA/VA client interfaces.
- **Vendor directory** (`vendor/`, 3022 files): all dependencies are vendored for reproducible builds.
- **Service discovery** via HashiCorp Consul (A-record and SRV record lookup for internal gRPC addresses).
- **Distributed tracing** via OpenTelemetry with OTLP/gRPC export (Jaeger in development).
- **Custom linter** (`linter/`) running zlint-based checks on precertificates before issuance.

---

## Database & Data Ownership

- **Datastore:** MariaDB 10.5 (accessed via ProxySQL 2.5.4 connection pooling), dialect `mysql`.
- **ORM:** `letsencrypt/borp` with `go-sql-driver/mysql`.
- **Migration tool:** `rubenv/sql-migrate`.

This service is the **sole owner** of two logical databases:

### `boulder_sa` schema (core ACME data)

| Table | Purpose |
|---|---|
| `authz2` | ACME authorizations (partitioned by id) |
| `blockedKeys` | Blocked public keys (by SPKI hash) |
| `certificateStatus` | Per-serial OCSP/revocation status (partitioned by id) |
| `certificates` | Issued end-entity certificates (DER + metadata, partitioned) |
| `certificatesPerName` | Rate-limit counters by eTLD+1 |
| `fqdnSets` | FQDN set fingerprints for issued certs (partitioned) |
| `incidents` | Incident registry (references incident-specific serial tables) |
| `issuedNames` | Reverse-indexed issued names for rate limiting (partitioned) |
| `keyHashToSerial` | Maps SPKI hash → certificate serial for key-based revocation |
| `newOrdersRL` | New-order rate-limit counters per registration |
| `orderFqdnSets` | FQDN set fingerprints per order (partitioned) |
| `orderToAuthz2` | Many-to-many: orders ↔ authorizations (partitioned) |
| `orders` | ACME orders |
| `precertificates` | Issued precertificates (DER, for CT and lint) |
| `registrations` | ACME account registrations |
| `requestedNames` | Names requested per order |
| `serials` | Certificate serial tracking |
| `crlShards` | CRL shard lease/state tracking (issuer × shard index) |

### `incidents_sa` schema (incident management)

| Table | Purpose |
|---|---|
| `incident_foo` | Example incident serial table (serial, registrationID, orderID, lastNoticeSent) |
| `incident_bar` | Example incident serial table (same structure) |

> These incident tables (`incident_foo`, `incident_bar`) are schema templates/examples; real incident tables are named per-incident and referenced dynamically via the `incidents.serialTable` column.

**Ownership boundary:** All database access is mediated exclusively through the SA gRPC service. No other component holds a direct database connection.

---

## Dependencies

### Runtime — internal gRPC service calls
- **VA** (`va/proto`): `PerformValidation`, `IsCAAValid`
- **CA / OCSPGenerator / CRLGenerator** (`ca/proto`): `IssuePrecertificate`, `IssueCertificateForPrecertificate`, `GenerateOCSP`, `GenerateCRL`
- **SA / StorageAuthority** (`sa/proto`): full read/write storage operations
- **Publisher** (`publisher/proto`): `SubmitToSingleCTWithResult`
- **Nonce Service** (`nonce/proto`): `Nonce`, `Redeem`
- **Akamai Purger** (`akamai/proto`): `Purge`
- **CRL Storer** (`crl/storer/proto`): `UploadCRL`

### Runtime — external infrastructure
| Dependency | Purpose |
|---|---|
| MariaDB 10.5 (via ProxySQL 2.5.4) | Primary relational datastore |
| Redis 6.2.7 (4 nodes: 2× OCSP, 2× rate-limits) | OCSP response cache, distributed rate-limit counters |
| AWS S3 (or S3-compatible) | CRL shard storage (`aws-sdk-go-v2/service/s3`) |
| HashiCorp Consul 1.15.4 | Internal service discovery (DNS / A-record) |
| Certificate Transparency logs | SCT submission via HTTP (`google/certificate-transparency-go`) |
| Akamai CDN | Cache purge for OCSP/CRL (via `AkamaiPurger` RPC) |
| PKCS#11 HSM / SoftHSM | Private key storage for CA signing (`pkcs11key`, `pkcs11`) |
| OTLP collector (Jaeger in dev) | Distributed trace export |

### Runtime — notable libraries
| Library | Purpose |
|---|---|
| `miekg/dns` | Custom DNS resolver for domain validation |
| `zmap/zlint` + `zmap/zcrypto` | Pre-issuance certificate linting |
| `weppos/publicsuffix-go` | eTLD+1 computation for rate limits |
| `golang/groupcache` | In-process caching |
| `go-jose/go-jose.v2` | JWK / JWS parsing for ACME |
| `titanous/rocacheck` | ROCA weak-key detection |
| `letsencrypt/validator` | Struct validation |
| `prometheus/client_golang` | Metrics exposition |
| `grpc-ecosystem/go-grpc-prometheus` | gRPC metrics interceptors |

### Build / test only
- `eggsampler/acme` — ACME test client
- `letsencrypt/challtestsrv` / `pebble-challtestsrv` — challenge test server
- `nxadm/tail` — log tailing in tests
- Python: `acme`, `cryptography`, `PyOpenSSL`, `requests` (integration tests); `matplotlib`, `numpy`, `pandas` (load-generator analysis)
- `golangci-lint`, `staticcheck`, `protoc-gen-go`, `protoc-gen-go-grpc`, `sql-migrate` (build pipeline tools)

---

## Deployment Model

### Container image
- Built from `test/boulder-tools/Dockerfile` (multi-stage: `buildpack-deps:focal-scm` base + Go 1.21.8 + Rust deps for `typos`).
- Image tag: `letsencrypt/boulder-tools:${BOULDER_TOOLS_TAG:-latest}`.
- Source tree is **bind-mounted** at `/boulder` (development mode); Go build cache mounted at `/.gocache`.
- `GOFLAGS=-mod=vendor` — all builds use the vendored dependency tree.
- Entrypoint: `test/entrypoint.sh` (launches all internal sub-services within the container for dev/CI).

### Ports exposed (host → container)
| Host Port | Container Port | Service |
|---|---|---|
| 4001 | 4001 | ACMEv2 HTTP API (WFE2) |
| 4002 | 4002 | OCSP responder |
| 4003 | 4003 | OCSP responder (secondary) |

Internal gRPC ports for sub-services (RA, CA, SA, VA, etc.) are not exposed to the host; they are resolved via Consul DNS within `bouldernet`.

### Networks (Docker Compose)
| Network | Subnet | Purpose |
|---|---|---|
| `bouldernet` | `10.77.77.0/24` | Primary inter-service network; boulder at `.77` |
| `integrationtestnet` | `10.88.88.0/24` | TLS-ALPN-01 / custom HTTP challenge servers |
| `redisnet` | `10.33.33.0/24` | Redis cluster (`.2`–`.5`) |
| `consulnet` | `10.55.55.0/24` | Consul agent at `.10` |

### Supporting services (Docker Compose)
- **`bmysql`** — MariaDB 10.5 with slow-query logging enabled.
- **`bproxysql`** — ProxySQL 2.5.4 at `boulder-proxysql:6033`, fronting MariaDB.
- **`bredis_1/2`** — Redis 6.2.7 for OCSP response caching.
- **`bredis_3/4`** — Redis 6.2.7 for rate-limit counters.
- **`bconsul`** — HashiCorp Consul 1.15.4 (dev agent) for service discovery.
- **`bjaeger`** — Jaeger 1.50 all-in-one for distributed tracing.

### Environment configuration
| Variable | Purpose |
|---|---|
| `BOULDER_CONFIG_DIR` | Path to JSON/YAML config files (default: `test/config`) |
| `FAKE_DNS` | IP for ACME challenge solver during development (`10.77.77.77`) |
| `GOCACHE` | Go build cache directory |
| `GOFLAGS` | Set to `-mod=vendor` |
| `GOEXPERIMENT` | Forwarded from host for experimental Go features |
| `BOULDER_TOOLS_TAG` | Docker image tag for CI pinning |

### Health / readiness endpoints
_Not determinable from code._
