---
repo: microservice1
spec_type: non_functional
commit: fc1d6e6323f710ab16190abe9550bf81c2b36f23
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: d184f820c48574690a1a274d278aa6d784a7a22a7d9a0daa5ad2cff964993cfa
generated_at: 2026-06-30T17:52:55.707749678+02:00
generator: specsync
---

## Performance

- The service is built on **Quarkus 3.37.0** with the reactive REST layer (`quarkus-rest`), which is built on Vert.x and provides non-blocking I/O by default, enabling high concurrency on a small thread pool.
- The application binds to `0.0.0.0:8080` (set via `JAVA_OPTS_APPEND` in all Dockerfiles).
- **JVM memory tuning** is delegated to the `run-java.sh` script in the UBI OpenJDK 21 base image. Heap defaults: initial = 25% of max (`JAVA_INITIAL_MEM_RATIO=25`), max = 50% of container memory limit (`JAVA_MAX_MEM_RATIO=50`), capped at 4096 MB initial (`JAVA_MAX_INITIAL_MEM=4096`). No explicit `-Xmx`/`-Xms` overrides are set in the manifests.
- A **native executable build profile** is provided (`Dockerfile.native`, `Dockerfile.native-micro`), which would yield lower startup latency and reduced memory footprint compared to JVM mode, but is not the default packaging.
- No explicit HTTP request timeouts, connection pool sizes, or caching configuration is present in `application.properties` or source code.
- All business logic (calculator, weather/forecast) executes in-memory with no I/O; latency is expected to be sub-millisecond (target).

_No throughput targets, SLA latency budgets, or explicit thread/connection-pool tuning are determinable from the code._

## Scalability

- The service is **stateless**: no session state, no shared in-memory cache, and no database or messaging dependency. All endpoints are independently invocable, making horizontal scaling straightforward.
- Containerisation is provided via four Dockerfiles (JVM, legacy-jar, native, native-micro), enabling deployment on any container orchestration platform.
- No replica counts, autoscaling policies (HPA/KEDA), or resource requests/limits (`resources.requests`/`resources.limits`) are defined in the provided manifests.
- No partitioning or sharding mechanisms are present; the workload is compute-only and uniformly distributable across replicas (target).

_Replica count and autoscaling configuration are not determinable from code._

## Security

- **No authentication or authorisation middleware** is present. None of the endpoints (`/calculator/*`, `/hello`, `/weather`, `/forecast`) are protected by any security filter, annotation (`@RolesAllowed`, `@Authenticated`), or Quarkus security extension.
- **No TLS/HTTPS configuration** is present; the service listens on plain HTTP port 8080.
- **Input validation** is minimal and application-level only:
  - `/calculator/divide`: checks divisor is non-zero; returns a structured error object rather than throwing.
  - `/weather` and `/weather/forecast`: null/empty `city` parameter check; `days` range check (1–7).
  - No framework-level bean validation (`@Valid`, `@NotNull`) is used.
- **Secrets handling**: no secrets, credentials, or API keys are present in configuration or source. The weather endpoint uses hard-coded mock data rather than calling an external API.
- The container runs as a **non-root user** (UID `185` in JVM images, UID `1001` in native images).
- No CORS, CSRF, rate-limiting, or input-sanitisation middleware is evident.

## Observability

- **Logging**: `org.jboss.logmanager.LogManager` is configured as the JUL logging manager via `JAVA_OPTS_APPEND` and Maven Surefire/Failsafe system properties, enabling Quarkus-native structured logging to stdout.
- **Metrics**: No metrics extension (`quarkus-micrometer`, `quarkus-smallrye-metrics`) is included in `pom.xml`. _Not determinable from code._
- **Tracing**: No tracing extension (`quarkus-opentelemetry`, `quarkus-smallrye-opentracing`) is included. _Not determinable from code._
- **Health/readiness endpoints**: No `quarkus-smallrye-health` extension is declared. Quarkus does not expose `/q/health` by default without this extension. _Not determinable from code._
- **Dev UI**: Available at `http://localhost:8080/q/dev/` in development mode only (`quarkus:dev`); not present in production packaging.
- No custom log statements, MDC enrichment, or correlation-ID propagation are visible in the source samples.

## Reliability

- **Error handling**: Application-level error responses are returned as structured JSON objects with an `error` field (e.g., division by zero, missing `city` parameter, invalid `days` range) rather than HTTP error status codes. No global exception mapper is evident.
- **Retries / circuit breakers**: No fault-tolerance extension (`quarkus-smallrye-fault-tolerance`) is present. No retry, circuit-breaker, fallback, or bulkhead annotations are used.
- **Idempotency**: All exposed endpoints use HTTP `GET` with query parameters and produce deterministic, side-effect-free responses, making them inherently idempotent.
- **Timeouts**: No client-side or server-side timeout configuration is set in `application.properties` or code.
- **External dependency resilience**: `quarkus-rest-client` and `quarkus-rest-client-jackson` are declared as dependencies, but no REST client interface or outbound call is implemented in the provided source. No resilience wrapper is applied to any outbound call.
- **Availability/recovery**: No liveness/readiness probes, graceful-shutdown configuration, or persistence layer are present, so failure modes are limited to process crash. Recovery is entirely dependent on the container orchestration platform restarting the container.
