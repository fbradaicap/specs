---
repo: loyaltymanagement
spec_type: non_functional
commit: a0321cd4386bc946a394b585316e19a23f985828
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 469b02de1c019b90b692b2f0f3c1092f8f3229e17169b313f078fb1f6acdcbef
generated_at: 2026-06-30T15:59:26.665857914+02:00
generator: specsync
---

## Performance

No explicit timeout, connection-pool, thread-pool, or caching configuration is present in `application.properties` or anywhere else in the snapshot. Spring Boot and Spring MVC defaults therefore apply:

- **Tomcat thread pool**: default core/max threads (10 / 200 in Spring Boot 3+).
- **HikariCP connection pool**: default pool size (10 connections) against the embedded H2 database.
- **Caching**: no caching layer configured.
- **HTTP timeouts**: no custom read/write/connection timeouts declared.

_Not determinable from code_ — no latency or throughput targets are specified.

## Scalability

- **Statelessness**: _Not determinable from code._ The service uses an embedded H2 in-memory database (runtime scope), which is inherently node-local and non-shareable, making horizontal scaling across multiple instances functionally impossible without a persistent, shared datastore.
- **Replica counts / autoscaling**: No Kubernetes manifests, Docker Compose files, or cloud-provider configuration are present in the snapshot. _Not determinable from code._
- **Partitioning / sharding**: _Not determinable from code._

> **Note**: The reliance on H2 in-memory storage is a hard constraint against horizontal scale-out in its current form.

## Security

- **AuthN / AuthZ**: No Spring Security dependency is declared; no authentication or authorisation middleware is configured. HTTP endpoints (when present) are effectively unauthenticated.
- **Transport security (TLS)**: No TLS/SSL properties are set in `application.properties`. The service runs on plain HTTP by default.
- **Secrets handling**: No secrets management (Vault, environment-variable injection, encrypted properties) is evident.
- **Input validation**: No `spring-boot-starter-validation` (Bean Validation / Hibernate Validator) dependency is declared; no validation annotations or filters are visible in the sampled source.
- **H2 Console**: `spring-boot-h2console` is included as a dependency. By default this exposes the H2 web console at `/h2-console` with no access control, which is a significant security exposure if the service is network-accessible.

## Observability

- **Logging**: Spring Boot default logging (Logback) is inherited from `spring-boot-starter-parent`; no custom logging configuration file (`logback-spring.xml`, etc.) is present. Log level and format are defaults.
- **Metrics**: No Micrometer or `spring-boot-starter-actuator` dependency is declared; no metrics endpoint is available.
- **Tracing**: No distributed tracing library (Micrometer Tracing, Sleuth, OpenTelemetry) is declared.
- **Health / readiness endpoints**: No `spring-boot-starter-actuator` is present; standard `/actuator/health` and `/actuator/info` endpoints are therefore **not** available.
- **API documentation**: `springdoc-openapi-starter-webmvc-ui` (v3.0.2) is included, exposing a Swagger UI at `/swagger-ui.html` and an OpenAPI JSON/YAML descriptor at `/v3/api-docs` (target — depends on endpoint implementation).

## Reliability

- **Resilience patterns**: No resilience4j, Spring Retry, or equivalent library is declared. No retry, circuit-breaker, bulkhead, or rate-limiter configuration is present.
- **Idempotency**: _Not determinable from code._
- **Failure handling**: No global exception handler (`@ControllerAdvice`) is visible in the sampled source.
- **Data durability**: H2 is configured in default in-memory mode; all data is lost on process restart, giving zero persistence durability.
- **Availability / recovery**: No deployment topology, health-check probe configuration, or liveness/readiness probe is defined. Single-instance, no-HA posture is implied by the current setup.
