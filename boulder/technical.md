---
repo: boulder
spec_type: technical
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:58:58.292168945+02:00
generator: specsync
---

## Tech Stack

- **Language:** Go 1.21 (primary; `go.mod` specifies `go 1.21`, build toolchain uses Go 1.21.8 per Dockerfile `ARG GO_VERSION: 1.21.8`)
- **Language:** Python (secondary; used in test tooling and load-generator scripts)
- **RPC framework:** gRPC (`google.golang.org/grpc v1.60.1`) with Protocol Buffers (`google.golang.org/protobuf v1.33.0`)
- **Database ORM/driver:** `github.com/letsencrypt/borp` (fork of go-borp ORM), `github.com/go-sql-driver/mysql v1.5.0` (MySQL/MariaDB driver), proxied via ProxySQL
- **Caching/rate-limit store:** Redis (`github.com/redis/go-redis/v9 v9.3.0`)
- **DNS library:** `github.com/miekg/dns v1.1.58`
- **Cryptography/HSM:** `github.com/letsencrypt/pkcs11key/v4`, `github.com/miekg/pkcs11 v1.1.1`, `golang.org/x/crypto v0.18.0`
- **Certificate linting:** `github.com/zmap/zlint/v3 v3.6.0`, `github.com/zmap/zcrypto`
- **ACME protocol:** `github.com/eggsampler/acme/v3 v3.5.0`, `gopkg.in/go-jose/go-jose.v2 v2.6.1`
- **Certificate Transparency:** `github.com/google/certificate-transparency-go v1.1.6`
- **Observability:** Prometheus (`github.com/prometheus/client_golang v1.15.1`), OpenTelemetry tracing (`go.opentelemetry.io/otel v1.24.0`, OTLP/gRPC exporter), Jaeger (via OTLP in dev)
- **Object storage:** AWS SDK v2 (`github.com/aws/aws-sdk-go-v2`, S3 client) for CRL upload
- **Service discovery:** HashiCorp Consul (DNS-based gRPC endpoint resolution)
- **Public suffix / domain validation:** `github.com/weppos/publicsuffix-go`, ROCA check `github.com/titanous/rocacheck`
- **Python test libs:** `acme>=2.0`, `cryptography`, `PyOpenSSL`, `requests`, `matplotlib`, `numpy`, `pandas`

---

## Architecture Patterns

Boulder is a **multi-component, microservice-style ACME Certificate Authority** implemented as a **monorepo**. All components communicate exclusively over **gRPC** (Protocol Buffers). The overall system follows a **layered / domain-separated architecture**:

| Component | Role |
|---|---|
| **WFE2** (Web Front End) | HTTP/ACME REST API entry point; translates ACME JSON requests into gRPC calls to the RA |
| **RA** (Registration Authority) | Orchestration layer; enforces ACME policy, rate limits, delegates to VA, CA, SA |
| **VA** (Validation Authority) | Performs ACME challenge validation (HTTP-01, DNS-01, TLS-ALPN-01) and CAA checks |
| **CA** (Certificate Authority) | Issues precertificates and final certificates; generates OCSP responses and CRLs; uses PKCS#11-backed keys |
| **SA** (Storage Authority) | Sole owner of persistent state; exposes a read-only and a read-write gRPC interface over MySQL via ProxySQL |
| **Publisher** | Submits precertificates/certificates to Certificate Transparency logs |
| **OCSP Responder** | Serves OCSP from Redis cache |
| **CRL Storer** | Uploads signed CRLs to S3-compatible object storage |
| **Nonce Service** | Issues and redeems anti-replay nonces (backed by Redis) |
| **Akamai Purger** | Purges CDN-cached OCSP/CRL URLs via Akamai API |
| **Observer** | Metrics/health monitoring component |

Key patterns:
- **Strict read/write separation in SA:** `StorageAuthorityReadOnly` vs. `StorageAuthority` gRPC services; read replicas can serve the read-only interface.
- **Streaming gRPC** used for bulk/large payloads (e.g., `GenerateCRL`, `UploadCRL`, `GetRevokedCerts`, `GetSerialsByAccount`, `SerialsForIncident`).
- **Event-driven via gRPC callbacks** (no message broker; the RA drives all orchestration synchronously).
- **Rate limiting** implemented in the `ratelimits/` package backed by Redis.
- **In-process linting** of issued certificates before issuance (`github.com/zmap/zlint`).
- **PKCS#11 HSM integration** for CA private key operations.
- **Vendored dependencies** (`vendor/` directory, `GOFLAGS: -mod=vendor`).

---

## Database & Data Ownership

**Datastore:** MySQL/MariaDB 10.5, accessed through **ProxySQL 2.5.4** (connection pooling and routing). Migrations managed by `github.com/rubenv/sql-migrate`.

**Two logical databases:**

### `boulder_sa` — Primary operational database (owned by the SA)

| Table | Purpose |
|---|---|
| `authz2` | ACME authorizations (partitioned by `id`) |
| `blockedKeys` | Blocked public keys (SPKI hash) |
| `certificateStatus` | Per-certificate OCSP/revocation status (partitioned by `id`) |
| `certificates` | Issued end-entity certificates and DER (partitioned by `id`) |
| `certificatesPerName` | Rate-limit counter: certs issued per eTLD+1 |
| `fqdnSets` | FQDN set hashes for exact-set rate limiting (partitioned by `id`) |
| `incidents` | Incident registry referencing per-incident serial tables |
| `issuedNames` | Reversed DNS name issuance log for rate limiting (partitioned by `id`) |
| `keyHashToSerial` | Maps public key hash → certificate serial |
| `newOrdersRL` | Rate-limit counter: new orders per registration |
| `orderFqdnSets` | FQDN set hashes scoped to orders (partitioned by `id`) |
| `orderToAuthz2` | Many-to-many: orders ↔ authorizations (partitioned) |
| `orders` | ACME orders |
| `precertificates` | Issued precertificates (lint copies) |
| `registrations` | ACME account registrations |
| `requestedNames` | Names requested in orders (for rate limiting) |
| `serials` | Serial number registry |
| `crlShards` | CRL shard lease tracking (issuer, shard index, lease window) |

### `incidents_sa` — Incident tracking database

| Table | Purpose |
|---|---|
| `incident_foo` | Example/template incident table: affected certificate serials with registration/order linkage |
| `incident_bar` | Second example/template incident table (same schema) |

> All data persistence is **exclusively owned and mediated by the SA**. No other component writes directly to the database.

---

## Dependencies

### Runtime — External Services

| Dependency | Type | Purpose |
|---|---|---|
| MySQL/MariaDB (via ProxySQL) | Database | Persistent storage for all ACME state |
| Redis (4 instances: 2 OCSP, 2 rate-limit) | Cache / rate-limit store | OCSP response cache; rate limit counters and nonce storage |
| HashiCorp Consul | Service discovery | gRPC service endpoint resolution via DNS A/SRV records |
| AWS S3 (or compatible) | Object storage | CRL shard upload (`crl/storer`) |
| Akamai CDN API | Third-party API | Cache purge for OCSP/CRL URLs |
| Certificate Transparency logs | Third-party APIs | SCT submission via `publisher` |
| PKCS#11 HSM / SoftHSM | Hardware/software | CA private key storage and signing |
| Jaeger (dev) | Tracing backend | Receives OTLP traces |

### Runtime — Key Go Libraries

| Library | Purpose |
|---|---|
| `google.golang.org/grpc` | All inter-service communication |
| `github.com/letsencrypt/borp` | MySQL ORM layer in SA |
| `github.com/go-sql-driver/mysql` | MySQL wire protocol driver |
| `github.com/redis/go-redis/v9` | Redis client |
| `github.com/miekg/dns` | DNS resolution for challenge validation |
| `github.com/miekg/pkcs11` / `letsencrypt/pkcs11key` | HSM/PKCS#11 interface |
| `github.com/google/certificate-transparency-go` | CT client and SCT parsing |
| `github.com/zmap/zlint/v3` | Pre-issuance certificate linting |
| `gopkg.in/go-jose/go-jose.v2` | JWK/JWS/JWT for ACME account keys |
| `github.com/weppos/publicsuffix-go` | Public suffix list for domain validation |
| `github.com/prometheus/client_golang` | Prometheus metrics exposition |
| `go.opentelemetry.io/otel` | Distributed tracing (OTLP/gRPC export) |
| `github.com/aws/aws-sdk-go-v2` + S3 | CRL upload to object storage |
| `github.com/titanous/rocacheck` | ROCA weak-key detection |
| `github.com/golang/groupcache` | In-process caching |
| `github.com/nxadm/tail` | Log file tailing (observer) |

### Build-time Only

| Tool | Purpose |
|---|---|
| `github.com/rubenv/sql-migrate` | Database migration runner |
| `google.golang.org/grpc/cmd/protoc-gen-go-grpc` | gRPC stub code generation |
| `google.golang.org/protobuf/cmd/protoc-gen-go` | Protobuf code generation |
| `github.com/golangci/golangci-lint` | Go linting |
| `honnef.co/go/tools/cmd/staticcheck` | Static analysis |
| Python: `acme`, `cryptography`, `PyOpenSSL`, `requests` | Integration test tooling |
| Python: `matplotlib`, `numpy`, `pandas` | Load test analysis/graphing |

---

## Deployment Model

### Container Image

Built from `test/boulder-tools/Dockerfile` (multi-stage):
- Stage 1 (`godeps`): installs Go 1.21.8, `protoc` plugins, `sql-migrate`, `pebble-challtestsrv`, `golangci-lint`, `staticcheck`
- Stage 2 (`rustdeps`): builds Rust-based tooling (`typos` spell checker)
- Final stage: `buildpack-deps:focal-scm` (Ubuntu Focal); copies Go toolchain, Rust binaries, Python requirements; configures rsyslog

Published image tag: `letsencrypt/boulder-tools:${BOULDER_TOOLS_TAG:-latest}`

### Orchestration

**Docker Compose** (`docker-compose.yml`) for local development and CI:

| Service | Image | Role |
|---|---|---|
| `boulder` | `letsencrypt/boulder-tools` | All Boulder components (launched via `test/entrypoint.sh`) |
| `bmysql` | `mariadb:10.5` | Primary database |
| `bproxysql` | `proxysql/proxysql:2.5.4` | Database proxy/load-balancer |
| `bredis_1–2` | `redis:6.2.7` | OCSP Redis cluster nodes |
| `bredis_3–4` | `redis:6.2.7` | Rate-limit Redis cluster nodes |
| `bconsul` | `hashicorp/consul:1.15.4` | Service discovery |
| `bjaeger` | `jaegertracing/all-in-one:1.50` | Distributed tracing (dev) |

### Networks

| Network | Subnet | Purpose |
|---|---|---|
| `bouldernet` | `10.77.77.0/24` | Primary inter-service network |
| `integrationtestnet` | `10.88.88.0/24` | Integration test challenge servers |
| `redisnet` | `10.33.33.0/24` | Redis cluster isolation |
| `consulnet` | `10.55.55.0/24` | Consul isolation |

### Ports (exposed on host)

| Port | Protocol | Purpose |
|---|---|---|
| `4001` | HTTP(S) | ACMEv2 API (WFE2) |
| `4002` | HTTP | OCSP responder |
| `4003` | HTTP | OCSP responder (secondary) |

All gRPC inter-service ports are internal to `bouldernet` only.

### Environment Configuration

| Variable | Purpose |
|---|---|
| `BOULDER_CONFIG_DIR` | Path to JSON config files (default: `test/config`) |
| `FAKE_DNS` | IP for ACME challenge DNS override |
| `GOCACHE` | Go build cache path |
| `GOFLAGS` | Set to `-mod=vendor` to use vendored dependencies |
| `GOEXPERIMENT` | Forwarded from host for experimental Go features |
| `BOULDER_TOOLS_TAG` | Docker image tag for CI pinning |

### Health / Readiness Endpoints

_Not determinable from code._
