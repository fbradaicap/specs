---
repo: pebble
spec_type: technical
commit: b37975ee86323dde963d5087b3103d2af34484fd
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9c647701705747ae93296249d39425b915371b7b8aaf4a2a3ebca6fb5d93a790
generated_at: 2026-06-30T14:54:09.090550209+02:00
generator: specsync
---

## Tech Stack

- **Language:** Go 1.21 (primary service implementation)
- **Language:** Python (test suite only, via `test/requirements.txt`)
- **Module path:** `github.com/letsencrypt/pebble/v2`
- **Key runtime libraries:**
  - `github.com/go-jose/go-jose/v4` v4.0.1 — JWS/JWK/JOSE operations (ACME request signing and verification)
  - `github.com/letsencrypt/challtestsrv` v1.3.2 — ACME challenge test server (bundled as both library and standalone binary)
  - `github.com/miekg/dns` v1.1.58 — DNS protocol implementation (DNS-01 challenge support, custom resolver)
  - `golang.org/x/crypto` v0.19.0 — supplemental cryptographic primitives
  - `golang.org/x/net` v0.21.0 — extended network primitives
- **Test-only Python libraries:** `acme`, `cryptography`, `josepy`, `pyOpenSSL`, `requests`
- **Build-only / indirect Go libraries:** `github.com/google/go-cmp`, `github.com/stretchr/testify`, `golang.org/x/mod`, `golang.org/x/sys`, `golang.org/x/tools`

---

## Architecture Patterns

**Layered / modular monolith with an embedded test-support sidecar.**

The codebase is organized into distinct internal packages that map to ACME CA subsystems:

| Package | Responsibility |
|---|---|
| `wfe` (Web Front End) | HTTP(S) request routing and ACME protocol handler; exposes both the ACME API and a Management API |
| `ca` | Certificate Authority: certificate issuance, chain construction, OCSP URL embedding |
| `va` (Validation Authority) | Domain control validation for HTTP-01, DNS-01, TLS-ALPN-01 challenges |
| `db` | In-memory store (`MemoryStore`) for all ACME objects (accounts, orders, authorizations, certificates) |
| `acme` | ACME data-type definitions shared across layers |
| `core` | Shared utility types |
| `cmd` | Entrypoint helpers (config loading, signal handling, error handling) |

The `pebble-challtestsrv` binary is a separate command (`cmd/pebble-challtestsrv`) that runs as a companion process providing controllable DNS/HTTP/TLS challenge responses; it communicates with test clients over an HTTP management API.

**Patterns evident:**
- **Layered architecture**: clear separation of protocol handling (`wfe`), business logic (`ca`, `va`), and storage (`db`).
- **Dependency injection**: `ca`, `va`, and `db` instances are constructed in `main` and injected into `wfe`.
- **Concurrent goroutines**: Management API server and ACME API server run concurrently in separate goroutines within the same process.
- **Strict mode flag**: runtime toggle for enforcing upcoming ACME spec changes, enabling forward-compatibility testing.

---

## Database & Data Ownership

- **Datastore:** In-memory only (`db.NewMemoryStore()`). No external database, no persistent storage, no migrations.
- **Data owned (in-memory):**
  - ACME Accounts
  - ACME Orders
  - ACME Authorizations
  - Issued Certificates
  - External Account Binding (EAB) keys
  - Blocked domain list
- **Persistence:** None — all state is lost on process restart. This is intentional; pebble is a test/development ACME CA.
- **Ownership boundary:** This service is the sole owner of its in-memory store. No shared datastore with other services.

---

## Dependencies

### Runtime dependencies (Go)

| Dependency | Purpose |
|---|---|
| `github.com/go-jose/go-jose/v4` | JWS signature verification and JWK handling for ACME request authentication |
| `github.com/letsencrypt/challtestsrv` | Challenge test server library (embedded in `pebble-challtestsrv` binary) |
| `github.com/miekg/dns` | DNS client and server for DNS-01 challenge validation and optional custom DNS resolver |
| `golang.org/x/crypto` | Additional cryptographic primitives (e.g., certificate operations) |
| `golang.org/x/net` | Extended network support (HTTP/2, IP socket options) |
| `golang.org/x/sys` | Low-level OS/syscall interface |

### Build/test-only dependencies (Go indirect)

| Dependency | Purpose |
|---|---|
| `github.com/google/go-cmp` | Deep equality comparison in tests |
| `github.com/stretchr/testify` | Test assertions |
| `golang.org/x/mod` | Go module utilities (tooling) |
| `golang.org/x/tools` | Go analysis tools (tooling) |

### Test-only dependencies (Python)

| Dependency | Purpose |
|---|---|
| `acme` | Python ACME client library for integration testing |
| `cryptography` | Cryptographic operations in Python test scripts |
| `josepy` | JOSE/JWK handling in Python tests |
| `pyOpenSSL` | TLS/certificate operations in Python tests |
| `requests` | HTTP client for Python integration tests |

### External service dependencies (runtime)
- **Custom DNS server** (optional): configurable via `-dnsserver` flag; pebble forwards DNS queries to this address during validation. In the compose setup this is `challtestsrv` at `10.30.50.3:8053`.
- **`pebble-challtestsrv`**: companion process (not a library call at runtime from `pebble` itself); pebble's VA makes outbound HTTP/DNS/TLS connections to addresses resolved through DNS, which the test environment routes to `challtestsrv`.

No third-party external APIs or message brokers are used.

---

## Deployment Model

### Container image

- **`Dockerfile.release`**: Multi-stage, multi-platform build. Accepts build args `APP` (defaults to `pebble`), `TARGETOS`, `TARGETARCH`. Uses `scratch` base for Linux and `mcr.microsoft.com/windows/nanoserver:ltsc2022` for Windows. Pre-built binaries are expected from a `dist-files` stage (not defined in the provided file — produced by a separate CI build step). Test configuration files from `./test/` are copied into `/test/` in the final image.
- **Published image:** `ghcr.io/letsencrypt/pebble:latest`
- **Companion image:** `ghcr.io/letsencrypt/pebble-challtestsrv:latest`

### Orchestration

- **`docker-compose.yml`** defines two services on a dedicated bridge network (`acmenet`, subnet `10.30.50.0/24`):

| Service | Image | IP | Ports |
|---|---|---|---|
| `pebble` | `ghcr.io/letsencrypt/pebble:latest` | `10.30.50.2` | `14000` (HTTPS ACME API), `15000` (HTTPS Management API) |
| `challtestsrv` | `ghcr.io/letsencrypt/pebble-challtestsrv:latest` | `10.30.50.3` | `8055` (HTTP Management API) |

- `pebble` is started with: `-config test/config/pebble-config.json -strict -dnsserver 10.30.50.3:8053`

### Ports (pebble process)

| Port | Protocol | Purpose |
|---|---|---|
| `14000` (configurable via `ListenAddress`) | HTTPS | ACME API (directory, newAccount, newOrder, etc.) |
| `15000` (configurable via `ManagementListenAddress`) | HTTPS | Management API (root CA cert retrieval, test controls) |

### Environment configuration

Configuration is primarily file-based (JSON config file via `-config` flag). Additionally, the following environment variables are read at startup:

| Variable | Effect |
|---|---|
| `PEBBLE_ALTERNATE_ROOTS` | Number of alternate root CA certificates to generate (integer ≥ 0) |
| `PEBBLE_CHAIN_LENGTH` | Intermediate certificate chain length (integer ≥ 0, default 1) |

### TLS

Both the ACME API and Management API are served over **TLS only** (`http.ListenAndServeTLS`). Certificate and private key paths are specified in the JSON config (`Certificate`, `PrivateKey` fields).

### Health / readiness endpoints

_Not determinable from code._ No dedicated `/healthz` or `/readyz` endpoints are evident in the provided source; the ACME directory endpoint (`https://<ListenAddress>/dir`) implicitly indicates liveness.
