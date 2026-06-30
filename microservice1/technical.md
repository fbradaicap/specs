---
repo: microservice1
spec_type: technical
commit: 67a1a3cf92762515763f4e7d8cea0cfc4eeb30c1
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: 50ee4e55544c2ceaf0a54f62861f4c7d0de3a524a44ec92fd862370af5bb0606
generated_at: 2026-06-30T17:38:56.381083786+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| Language | Java 21 (`maven.compiler.release=21`) |
| Runtime | JVM (OpenJDK 21) or GraalVM native executable |
| Framework | Quarkus 3.37.0 |
| REST layer | `quarkus-rest` (Jakarta REST / JAX-RS) |
| DI container | `quarkus-arc` (Quarkus CDI implementation) |
| Build tool | Apache Maven (Maven Wrapper included), `quarkus-maven-plugin` 3.37.0 |
| Test libraries | `quarkus-junit` (runtime scope: test), `rest-assured` (runtime scope: test) |
| Compiler plugin | `maven-compiler-plugin` 3.15.0 |
| Test runner plugins | `maven-surefire-plugin` 3.5.6, `maven-failsafe-plugin` 3.5.6 |

## Architecture Patterns

- **Layered REST API (single-resource):** The service follows a minimal, flat layered structure with a single Jakarta REST resource class acting as both the controller and the business logic layer — no separate service or repository layers are present.
- **CDI-managed bean:** `HelloResource` is managed by Quarkus Arc (CDI), enabling dependency injection if needed in future extensions.
- **Stateless request/response:** All endpoints are stateless; no session or persistent state is maintained.

Key internal component:

| Component | Role |
|---|---|
| `com.example.HelloResource` | Single JAX-RS resource exposing `GET /hello`; returns plain text or JSON |

## Database & Data Ownership

This service owns **no datastore**. No database drivers, ORM frameworks, or migration tools are declared in `pom.xml`. No DB tables or models were detected. `application.properties` is empty, confirming no datasource configuration.

## Dependencies

### Runtime dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `io.quarkus:quarkus-rest` | Runtime | JAX-RS/Jakarta REST implementation for HTTP endpoints |
| `io.quarkus:quarkus-arc` | Runtime | CDI dependency injection container |
| `io.quarkus.platform:quarkus-bom` 3.37.0 | BOM (dependency management) | Quarkus platform bill of materials |

### Test-scoped dependencies

| Dependency | Type | Purpose |
|---|---|---|
| `io.quarkus:quarkus-junit` | Test | Quarkus test framework (`@QuarkusTest`, `@QuarkusIntegrationTest`) |
| `io.rest-assured:rest-assured` | Test | HTTP integration/acceptance test DSL |

### Build plugins

| Plugin | Version | Purpose |
|---|---|---|
| `quarkus-maven-plugin` | 3.37.0 | Quarkus build, dev mode, native compilation |
| `maven-compiler-plugin` | 3.15.0 | Java compilation |
| `maven-surefire-plugin` | 3.5.6 | Unit test execution |
| `maven-failsafe-plugin` | 3.5.6 | Integration test execution |

No external services, message brokers, caches, or third-party APIs are consumed.

## Deployment Model

### Container images

Four Dockerfile variants are provided under `src/main/docker/`:

| Dockerfile | Base image | Mode |
|---|---|---|
| `Dockerfile.jvm` | `ubi9/openjdk-21-runtime:1.24` | JVM — layered fast-jar (`quarkus-run.jar`) |
| `Dockerfile.legacy-jar` | `ubi9/openjdk-21-runtime:1.24` | JVM — legacy über-jar |
| `Dockerfile.native` | `ubi9/ubi-minimal:9.7` | Native executable (no JVM) |
| `Dockerfile.native-micro` | `ubi9-quarkus-micro-image:2.0` | Native executable (minimal image) |

All variants:
- **Expose port:** `8080`
- **Run as user:** non-root (`185` for JVM images, `1001` for native images)
- **Entrypoint (JVM):** `/opt/jboss/container/java/run/run-java.sh` with `JAVA_APP_JAR=/deployments/quarkus-run.jar`
- **Entrypoint (native):** `./application -Dquarkus.http.host=0.0.0.0`
- **JVM tuning:** Handled via `run-java.sh` environment variables (`JAVA_OPTS_APPEND`, `JAVA_MAX_MEM_RATIO`, etc.)

### Build

```bash
# JVM mode
./mvnw package
docker build -f src/main/docker/Dockerfile.jvm -t quarkus/hello-quarkus-jvm .

# Native mode
./mvnw package -Dnative
docker build -f src/main/docker/Dockerfile.native -t quarkus/hello-quarkus .
```

### Orchestration

_Not determinable from code._ No Kubernetes manifests, Helm charts, or Docker Compose files are present in the repository.

### Environment configuration

| Variable | Default / Note |
|---|---|
| `JAVA_OPTS_APPEND` | `-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager` |
| `LANGUAGE` | `en_US:en` |

`application.properties` is empty; no additional application-level configuration is defined.

### Health / readiness endpoints

Quarkus exposes a Dev UI at `http://localhost:8080/q/dev/` in dev mode. Dedicated health/readiness endpoints (`/q/health`) are _not determinable from code_ — the `quarkus-smallrye-health` extension is not declared in `pom.xml`.
