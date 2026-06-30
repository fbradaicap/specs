---
repo: microservice1
spec_type: technical
commit: fc1d6e6323f710ab16190abe9550bf81c2b36f23
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: d184f820c48574690a1a274d278aa6d784a7a22a7d9a0daa5ad2cff964993cfa
generated_at: 2026-06-30T17:52:55.707749678+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| Language | Java 21 (source/release target: `21`) |
| Runtime framework | Quarkus 3.37.0 |
| REST layer | `quarkus-rest` (Jakarta REST / JAX-RS reactive stack) |
| JSON serialisation | `quarkus-rest-jackson` / `quarkus-rest-client-jackson` |
| HTTP client | `quarkus-rest-client` (MicroProfile REST Client) |
| DI container | `quarkus-arc` (CDI 4.x) |
| Build tool | Apache Maven (wrapper included), `quarkus-maven-plugin` 3.37.0 |
| Base image (JVM) | `registry.access.redhat.com/ubi9/openjdk-21-runtime:1.24` |
| Base image (native) | `registry.access.redhat.com/ubi9/ubi-minimal:9.7` / `quay.io/quarkus/ubi9-quarkus-micro-image:2.0` |
| Test frameworks | `quarkus-junit`, REST-Assured, `maven-surefire-plugin` 3.5.6, `maven-failsafe-plugin` 3.5.6 |

## Architecture Patterns

**Style:** Monolithic REST API — a single deployable unit exposing multiple resource endpoints with no asynchronous or event-driven elements detected.

**Layered structure (flat):**
- **Resource/controller layer** — three JAX-RS resource classes handle all HTTP concerns and inline business logic directly (no separate service or repository layers are present).

**Key internal components:**

| Component | Path | Responsibility |
|---|---|---|
| `CalculatorResource` | `/calculator` | Arithmetic operations (add, subtract, multiply, divide) via query parameters |
| `HelloResource` | `/hello` | Probe / greeting endpoint returning plain text or JSON |
| `WeatherResource` | `/weather` | Mock weather current-conditions and multi-day forecast |

**Notable patterns:**
- Static inner classes (`CalculationResult`, `WeatherInfo`, `WeatherForecast`, `DayForecast`) used as plain response DTOs serialised to JSON via Jackson.
- Weather data is fully mocked (hard-coded `switch` block); no external weather API client is wired at runtime despite `quarkus-rest-client` being on the classpath.
- Supports both JVM and GraalVM native-image packaging modes via Maven profiles.

## Database & Data Ownership

This service owns **no datastore**. No database drivers, ORM frameworks, datasource configuration, or schema migrations are present. All state is ephemeral and computed on-request from hard-coded or in-memory logic.

## Dependencies

### Runtime
| Dependency | Type | Purpose |
|---|---|---|
| `quarkus-rest` | Framework | Jakarta REST endpoint routing and HTTP server |
| `quarkus-arc` | Framework | CDI dependency injection |
| `quarkus-rest-jackson` | Library | JSON serialisation/deserialisation for REST responses |
| `quarkus-rest-client` | Library | MicroProfile REST Client (declared but no concrete client interface detected in source) |
| `quarkus-rest-client-jackson` | Library | Jackson integration for outbound REST client calls |

### Build / Test only
| Dependency | Scope | Purpose |
|---|---|---|
| `quarkus-junit` | `test` | Quarkus test harness (`@QuarkusTest`, `@QuarkusIntegrationTest`) |
| `rest-assured` | `test` | HTTP integration/black-box test assertions |
| `maven-compiler-plugin` 3.15.0 | build | Java compilation with `-parameters` flag enabled |
| `maven-surefire-plugin` 3.5.6 | build | Unit test execution |
| `maven-failsafe-plugin` 3.5.6 | build | Integration test execution (bound to `integration-test`/`verify` goals) |

No external third-party SaaS APIs, message brokers, or caches are consumed at runtime.

## Deployment Model

**Build:**
```
./mvnw package                          # fast-jar (default)
./mvnw package -Dquarkus.package.jar.type=uber-jar   # über-jar
./mvnw package -Dnative                 # GraalVM native binary
```

**Container images — four Dockerfiles provided:**

| Dockerfile | Mode | Base image |
|---|---|---|
| `Dockerfile.jvm` | JVM, fast-jar (layered) | `ubi9/openjdk-21-runtime:1.24` |
| `Dockerfile.legacy-jar` | JVM, legacy über-jar | `ubi9/openjdk-21-runtime:1.24` |
| `Dockerfile.native` | GraalVM native | `ubi9/ubi-minimal:9.7` |
| `Dockerfile.native-micro` | GraalVM native (minimal) | `ubi9-quarkus-micro-image:2.0` |

**Ports:** `8080` (HTTP) exposed in all Dockerfiles. Optional debug port `5005` documented but not exposed by default.

**Key environment variables (JVM images):**

| Variable | Default / Purpose |
|---|---|
| `JAVA_OPTS_APPEND` | `-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager` |
| `JAVA_APP_JAR` | `/deployments/quarkus-run.jar` |
| `JAVA_DEBUG` | Remote debug toggle (default off) |
| `JAVA_MAX_MEM_RATIO` | Heap ceiling ratio relative to container memory limit (default 50%) |

**Orchestration:** _Not determinable from code._ No Kubernetes manifests, Helm charts, or Docker Compose files are present in the repository.

**Health/readiness endpoints:** Quarkus ships built-in liveness/readiness probes at `/q/health/live` and `/q/health/ready` when `quarkus-smallrye-health` is on the classpath; however, that extension is **not** declared in `pom.xml`, so dedicated health endpoints are _not determinable from code._

**Application configuration:** `src/main/resources/application.properties` is present but empty — all settings use Quarkus defaults (HTTP on `0.0.0.0:8080`).
