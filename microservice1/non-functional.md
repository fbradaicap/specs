---
repo: microservice1
spec_type: non_functional
commit: 67a1a3cf92762515763f4e7d8cea0cfc4eeb30c1
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: 50ee4e55544c2ceaf0a54f62861f4c7d0de3a524a44ec92fd862370af5bb0606
generated_at: 2026-06-30T17:38:56.381083786+02:00
generator: specsync
---

## Performance

No explicit latency targets, throughput limits, or timeout configurations are present in `application.properties` or the source code. Quarkus 3.37.0 defaults apply.

- **HTTP port:** 8080 (all interfaces, set via `JAVA_OPTS_APPEND` in Dockerfiles).
- **JVM heap:** Managed dynamically by the `run-java.sh` script in the UBI OpenJDK 21 base image. Default `JAVA_MAX_MEM_RATIO=50` (50 % of container memory limit) and `JAVA_INITIAL_MEM_RATIO=25` apply unless overridden at runtime; no explicit `-Xmx`/`-Xms` values are configured.
- **GC:** Defaults to `-XX:+UseParallelGC` via `run-java.sh`; no custom GC flags are set.
- **Connection/thread pools:** Quarkus REST (Vert.x-based) defaults — no custom pool sizes configured.
- **Caching:** _Not determinable from code._
- **Native executable profile** is available (`mvn package -Dnative`), which would reduce startup time and memory footprint, but is not the default build.

## Scalability

- **Statelessness:** The single resource class (`HelloResource`) holds no instance state, making the service inherently stateless and horizontally scalable (target).
- **Replica counts / autoscaling:** No Kubernetes manifests, Helm charts, or autoscaling annotations are present. _Not determinable from code._
- **Horizontal scaling approach:** Containerisation is fully supported via four provided Dockerfiles (JVM, legacy-jar, native, native-micro); container-aware heap sizing is built into the base image. Horizontal scaling is architecturally possible (target) but no orchestration configuration is provided.
- **Partitioning/sharding:** _Not determinable from code._ No database or messaging layer exists.

## Security

- **AuthN/AuthZ:** No authentication or authorisation mechanisms are configured. The `/hello` endpoint is publicly accessible with no security middleware present.
- **Transport security (TLS):** Not configured. HTTP only on port 8080; no TLS/HTTPS settings appear in `application.properties` or Dockerfiles.
- **Secrets handling:** No secrets, credentials, or vault integrations are referenced anywhere in the codebase.
- **Input validation:** The sole endpoint (`GET /hello`) accepts no user-supplied input; no validation framework is in use.
- **Container hardening:** Dockerfiles run the process as a non-root user (UID `185` for JVM images, UID `1001` for native images), which is a positive security baseline.
- **Security dependencies/middleware:** None detected (`quarkus-oidc`, `quarkus-security`, `quarkus-smallrye-jwt`, etc. are absent).

## Observability

- **Logging:** JBoss LogManager (`org.jboss.logmanager.LogManager`) is configured as the Java logging manager in all Dockerfiles and in Maven Surefire/Failsafe configuration. Log output goes to stdout by default (no file appender configured).
- **Metrics:** No metrics extension (e.g., `quarkus-micrometer`, `quarkus-smallrye-metrics`) is present. _Not determinable from code._
- **Tracing:** No tracing extension (e.g., `quarkus-opentelemetry`) is present. _Not determinable from code._
- **Health/readiness endpoints:** No `quarkus-smallrye-health` dependency is included. Quarkus built-in liveness/readiness probes (`/q/health`) are not available in this build.
- **Dev UI:** Available at `http://localhost:8080/q/dev/` in `quarkus:dev` mode only; not exposed in production images.

## Reliability

- **Retries / circuit breakers:** No fault-tolerance extension (e.g., `quarkus-smallrye-fault-tolerance`) is present; no retry, circuit-breaker, or bulkhead annotations are used.
- **Timeouts:** No request timeout or idle-connection timeout is configured.
- **Idempotency:** The single `GET /hello` endpoint is inherently idempotent by HTTP semantics.
- **Failure handling:** No explicit exception mappers or error-handling logic beyond Quarkus defaults are implemented.
- **Availability / recovery:** No readiness/liveness probes, no persistent state, and no dependency on external systems; recovery is purely a function of container restart policy, which is not configured in the provided manifests. _Not determinable from code._
