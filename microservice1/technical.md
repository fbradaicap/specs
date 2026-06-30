---
repo: microservice1
spec_type: technical
commit: 954d85981a924722137ae7edea41a6ffbf5b2444
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d7c1d6d7e1a8acbea7e48f4ff300b586a4fb468527117a2559d0a34869f35a63
generated_at: 2026-06-30T16:00:13.254039078+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|-----------|--------|
| Language | Java 21 (compiler release target: 21) |
| Runtime | JVM (OpenJDK 21) or GraalVM native executable |
| Framework | Quarkus 3.37.0 |
| REST layer | `quarkus-rest` (Jakarta REST / JAX-RS, reactive) |
| DI container | `quarkus-arc` (CDI-based) |
| Build tool | Apache Maven (Maven Wrapper included) with `quarkus-maven-plugin` 3.37.0 |
| Test libraries | `quarkus-junit` (unit + integration), `rest-assured` (HTTP assertions) |
| Base image (JVM) | `registry.access.redhat.com/ubi9/openjdk-21-runtime:1.24` |
| Base image (native) | `registry.access.redhat.com/ubi9/ubi-minimal:9.7` / `quay.io/quarkus/ubi9-quarkus-micro-image:2.0` |

## Architecture Patterns

The service follows a **simple single-layer REST API** pattern with no additional architectural complexity:

- **REST API**: A single JAX-RS resource class (`HelloResource`) is registered at the path `/hello` and returns a plain-text response to HTTP GET requests.
- **CDI-managed beans**: Quarkus Arc provides the dependency injection container; the resource is a CDI-managed bean by default.
- **No layering beyond the resource**: There are no service, repository, or domain layers present — the resource method directly returns the response string.
- **Test strategy**: Unit tests run via `@QuarkusTest` (starts a test instance of the application); integration tests run via `@QuarkusIntegrationTest` against the packaged artefact (JVM or native).

## Database & Data Ownership

This service owns **no datastores**. No database drivers, ORM frameworks, migration tools, or schema definitions are present in the build manifests or source tree. No DB tables or models were detected.

## Dependencies

### Runtime dependencies

| Dependency | Purpose |
|------------|---------|
| `io.quarkus:quarkus-rest` | Jakarta REST (JAX-RS) layer; exposes HTTP endpoints |
| `io.quarkus:quarkus-arc` | CDI dependency injection container |
| `io.quarkus.platform:quarkus-bom` 3.37.0 | BOM for consistent Quarkus dependency versions |

### Test-scope dependencies

| Dependency | Purpose |
|------------|---------|
| `io.quarkus:quarkus-junit` | Quarkus test framework (`@QuarkusTest`, `@QuarkusIntegrationTest`) |
| `io.rest-assured:rest-assured` | HTTP-level assertion library for endpoint tests |

### Build-time plugins

| Plugin | Version | Purpose |
|--------|---------|---------|
| `quarkus-maven-plugin` | 3.37.0 | Quarkus build lifecycle (packaging, dev mode) |
| `maven-compiler-plugin` | 3.15.0 | Java compilation |
| `maven-surefire-plugin` | 3.5.6 | Unit test execution |
| `maven-failsafe-plugin` | 3.5.6 | Integration test execution |

No external service calls, message brokers, caches, or third-party APIs are present.

## Deployment Model

### Container images
Four Dockerfiles are provided for different packaging modes:

| Dockerfile | Mode | Base image |
|------------|------|------------|
| `Dockerfile.jvm` | JVM, layered fast-jar | `ubi9/openjdk-21-runtime:1.24` |
| `Dockerfile.legacy-jar` | JVM, legacy über-jar | `ubi9/openjdk-21-runtime:1.24` |
| `Dockerfile.native` | GraalVM native executable | `ubi9/ubi-minimal:9.7` |
| `Dockerfile.native-micro` | GraalVM native, micro image | `ubi9-quarkus-micro-image:2.0` |

All images **expose port `8080`** and run as non-root (user `185` for JVM images, user `1001` for native images).

The JVM image entrypoint uses Red Hat's `run-java.sh` script with the following fixed JVM options:
```
-Dquarkus.http.host=0.0.0.0
-Djava.util.logging.manager=org.jboss.logmanager.LogManager
```

### Build
```bash
./mvnw package                          # JVM fast-jar
./mvnw package -Dnative                 # Native executable
```

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 8080 | HTTP | Application HTTP listener |

### Environment configuration
`src/main/resources/application.properties` is empty — no application-level configuration properties are defined. Runtime tuning is available via standard Quarkus environment variables (`JAVA_OPTS_APPEND`, `JAVA_MAX_MEM_RATIO`, etc.) as documented in the Dockerfiles.

### Orchestration
_Not determinable from code._ No Kubernetes manifests, Helm charts, or Docker Compose files are present in the repository.

### Health / readiness endpoints
Quarkus's built-in health endpoints (`/q/health`, `/q/health/live`, `/q/health/ready`) are available in dev mode via the Dev UI at `http://localhost:8080/q/dev/`, but the `quarkus-smallrye-health` extension is not declared as a dependency, so **dedicated health/readiness endpoints are not configured**.
