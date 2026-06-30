---
repo: loyaltymanagement
spec_type: technical
commit: a0321cd4386bc946a394b585316e19a23f985828
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 469b02de1c019b90b692b2f0f3c1092f8f3229e17169b313f078fb1f6acdcbef
generated_at: 2026-06-30T15:59:26.665857914+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| Language | Java 21 |
| Runtime | JVM (Java 21) |
| Framework | Spring Boot 4.0.6 (via `spring-boot-starter-parent`) |
| Web layer | Spring MVC (`spring-boot-starter-webmvc`) |
| Persistence | Spring Data JPA (`spring-boot-starter-data-jpa`) |
| Database (runtime) | H2 in-memory database (`com.h2database:h2`, runtime scope) |
| API documentation | springdoc-openapi 3.0.2 (`springdoc-openapi-starter-webmvc-ui`) — exposes Swagger UI |
| Code generation | Lombok (compile-time annotation processor; excluded from final artifact) |
| Build tool | Apache Maven (via Maven Wrapper, `.mvn/`) |
| Test framework | Spring Boot Test slices for JPA and MVC (test scope) |

## Architecture Patterns

The service follows a **classic layered (n-tier) REST API** pattern typical of Spring Boot applications:

- **Presentation layer** — Spring MVC controllers intended to expose HTTP REST endpoints (no controller source files are present at this commit, indicating the service is in early/scaffold stage).
- **Persistence layer** — Spring Data JPA repositories backed by an H2 in-memory datastore.
- **API documentation** — Swagger UI auto-generated via springdoc-openapi, reachable at the standard `/swagger-ui.html` or `/swagger-ui/index.html` path.
- **H2 console** — The `spring-boot-h2console` dependency enables the browser-based H2 console for development-time inspection.

No CQRS, event-driven, or hexagonal patterns are evident at this stage. The application is a single deployable unit (monolith-in-a-jar) with no async messaging.

## Database & Data Ownership

| Item | Detail |
|---|---|
| Datastore type | H2 in-memory relational database (runtime dependency) |
| Persistence mechanism | Spring Data JPA / Hibernate (auto-DDL expected) |
| Owned tables / models | _Not determinable from code._ — No entity classes or migration scripts are present at head commit |
| Ownership boundary | This service is the sole owner of its embedded H2 instance; no shared datastore is configured |

> **Note:** Because H2 is in-memory and no external datasource URL is defined in `application.properties`, all data is ephemeral and lost on restart. A persistent RDBMS datasource has not yet been configured.

## Dependencies

### Runtime dependencies

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-webmvc` | Embedded Tomcat + Spring MVC for HTTP request handling |
| `spring-boot-starter-data-jpa` | JPA/Hibernate ORM + Spring Data repository abstraction |
| `spring-boot-h2console` | Browser-based H2 administration console |
| `com.h2database:h2` | H2 in-memory RDBMS |
| `springdoc-openapi-starter-webmvc-ui:3.0.2` | OpenAPI 3 spec generation and Swagger UI |

### Build / compile-time dependencies

| Dependency | Purpose |
|---|---|
| `org.projectlombok:lombok` | Boilerplate code generation (getters, builders, etc.) — excluded from packaged JAR |

### Test dependencies

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-data-jpa-test` | JPA slice testing (`@DataJpaTest`) |
| `spring-boot-starter-webmvc-test` | MVC slice testing (`@WebMvcTest`) |

### External service / broker / third-party API dependencies

_Not determinable from code._ — No outbound HTTP clients, message broker configuration, or third-party API integrations are present at head commit.

## Deployment Model

| Aspect | Detail |
|---|---|
| Build artifact | Executable fat JAR produced by `spring-boot-maven-plugin` |
| Containerisation | _Not determinable from code._ — No `Dockerfile` or container image definition is present |
| Orchestration | _Not determinable from code._ — No Kubernetes manifests, Helm charts, or Docker Compose files are present |
| Exposed port | Spring Boot default `8080` (no override in `application.properties`) |
| H2 console path | `/h2-console` (default, enabled by `spring-boot-h2console`) |
| Swagger UI path | `/swagger-ui/index.html` (springdoc-openapi default) |
| Health / readiness endpoints | _Not determinable from code._ — Spring Boot Actuator is not listed as a dependency; no explicit health endpoint is configured |
| Environment configuration | Only `spring.application.name=loyalty-management` is set; all other configuration relies on Spring Boot defaults |
