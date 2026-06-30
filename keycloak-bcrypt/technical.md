---
repo: keycloak-bcrypt
spec_type: technical
commit: 9b2c39431b99ae6a19eeda033dd8acba5c772bb7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 34b481fd1be09917de008c5a9821d6465282d96dc7af18c5b209ab170f965a82
generated_at: 2026-06-30T14:54:03.729924115+02:00
generator: specsync
---

## Tech Stack

- **Language:** Java 17 (source and target compatibility set to `JavaVersion.VERSION_17`)
- **Build tool:** Gradle 7.6 (Kotlin DSL — `build.gradle.kts`)
- **Runtime base image:** [quay.io/keycloak/keycloak](https://quay.io/repository/keycloak/keycloak) (version parameterised via `${keycloak_version}` build arg)
- **Key runtime library:**
  - `at.favre.lib:bcrypt:0.10.2` — BCrypt hashing implementation (bundled into the fat JAR)
- **Compile-only (provided by Keycloak at runtime):**
  - `org.keycloak:keycloak-common`, `keycloak-core`, `keycloak-server-spi`, `keycloak-server-spi-private` (version externalised as `dependency.keycloak.version`)
  - `org.jboss.logging:jboss-logging:3.4.1.Final`
- **Test libraries:**
  - `org.junit.jupiter:junit-jupiter-api:5.8.2` / `junit-jupiter-engine:5.8.2`

---

## Architecture Patterns

This is a **Keycloak SPI (Service Provider Interface) plugin**, not a standalone microservice. It follows the **provider/factory pattern** mandated by the Keycloak extension model:

- **`BCryptPasswordHashProviderFactory`** — implements `PasswordHashProviderFactory`; registered via Keycloak's SPI discovery mechanism (Java ServiceLoader). Acts as a factory, instantiating the provider per session. Provider ID: `bcrypt`, default cost factor: `10`.
- **`BCryptPasswordHashProvider`** — implements `PasswordHashProvider`; handles encoding, policy checking, and verification of BCrypt password hashes. Verification supports both `2Y` (primary, with truncation strategy for long passwords) and `2A` (Blowfish fallback) variants.
- The plugin is packaged as a **fat/uber JAR** (runtime classpath dependencies are merged in at build time, excluding conflicting `META-INF` signatures), dropped into Keycloak's `providers/` directory.
- No HTTP endpoints, message consumers, or independent process — execution is entirely driven by Keycloak's internal credential-management lifecycle.

---

## Database & Data Ownership

This service **owns no data**. It introduces no database schema, tables, or migrations. All credential storage is delegated entirely to Keycloak's own datastore. The BCrypt salt is embedded within the encoded password string (a BCrypt convention) and stored by Keycloak alongside the credential record.

For local development/testing, the `docker-compose.yml` provisions a **PostgreSQL 5432** instance used exclusively by Keycloak (database: `keycloak`); this is infrastructure for the host Keycloak process, not owned by this plugin.

---

## Dependencies

### Runtime (bundled in JAR)
| Dependency | Purpose |
|---|---|
| `at.favre.lib:bcrypt:0.10.2` | BCrypt hashing and verification |

### Compile-only / Provided at runtime by Keycloak
| Dependency | Purpose |
|---|---|
| `org.keycloak:keycloak-server-spi` | `PasswordHashProviderFactory` SPI interface |
| `org.keycloak:keycloak-server-spi-private` | Private SPI extensions |
| `org.keycloak:keycloak-core` | `PasswordCredentialModel`, `PasswordPolicy` |
| `org.keycloak:keycloak-common` | Shared Keycloak utilities |
| `org.jboss.logging:jboss-logging:3.4.1.Final` | Logging façade used by Keycloak ecosystem |

### Build-only
| Dependency | Purpose |
|---|---|
| `org.junit.jupiter:junit-jupiter-api/engine:5.8.2` | Unit testing |
| `gradle:jdk21` (Docker build stage) | Compilation environment |

### External service dependencies (docker-compose, dev only)
| Service | Purpose |
|---|---|
| PostgreSQL (official image) | Keycloak backing store — dev/test only |

---

## Deployment Model

### Container Image
- **Multi-stage Dockerfile:**
  1. **Build stage:** `gradle:${gradle_version}` (default tag `jdk21`); copies source, runs `gradle assemble`.
  2. **Final stage:** `quay.io/keycloak/keycloak:${keycloak_version}`; copies the assembled JAR into `/opt/keycloak/providers/`.
- Both `gradle_version` and `keycloak_version` are parameterised `ARG` values supplied at `docker build` time.
- Published image: `gleroy/keycloak-bcrypt`

### Orchestration (docker-compose — dev/test only)
| Setting | Value |
|---|---|
| Exposed port | `8443` (HTTPS, mapped to host `8443`) |
| Keycloak startup command | `start --auto-build --db=postgres` |
| Database | PostgreSQL at `jdbc:postgresql://db/keycloak` |
| TLS keystore | `./conf/server.keystore` → `/opt/keycloak/conf/server.keystore` |
| Admin credentials | `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` (env vars) |
| Hostname | `KC_HOSTNAME` (env var) |

### Environment Variables (Keycloak runtime)
| Variable | Purpose |
|---|---|
| `KEYCLOAK_ADMIN` | Admin username |
| `KEYCLOAK_ADMIN_PASSWORD` | Admin password |
| `KC_HOSTNAME` | Advertised hostname |
| `KC_DB_URL` | JDBC connection URL |
| `KC_DB_USERNAME` / `KC_DB_PASSWORD` | Database credentials |

### Health / Readiness Endpoints
_Not determinable from code._ (Delegated entirely to Keycloak's built-in health probes.)

### Kubernetes / Helm
_Not determinable from code._ No Kubernetes or Helm manifests are present in the repository.
