---
repo: microservice1
spec_type: non_functional
commit: 954d85981a924722137ae7edea41a6ffbf5b2444
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d7c1d6d7e1a8acbea7e48f4ff300b586a4fb468527117a2559d0a34869f35a63
generated_at: 2026-06-30T16:00:13.254039078+02:00
generator: specsync
---

## Performance

No explicit latency targets, throughput limits, timeout configurations, connection pool settings, or caching directives are present in `application.properties` or any other configuration file. The service runs on Quarkus 3.37.0 with the `quarkus-rest` (RESTEasy Reactive) extension, which uses a non-blocking I/O model backed by Vert.x; default Quarkus worker-thread and event-loop sizing applies but no overrides are configured.

- **HTTP port:** 8080 (default Quarkus)
- **JVM heap:** Dynamically sized at runtime via the `run-java.sh` script; default `JAVA_MAX_MEM_RATIO=50` (50% of container memory limit) and `JAVA_INITIAL_MEM_RATIO=25` apply unless overridden through environment variables.
- **GC:** Defaults to `-XX:+UseParallelGC` unless `GC_CONTAINER_OPTIONS` is set.
- **Native build:** Supported via GraalVM / container-based native build; would eliminate JVM warm-up overhead (target).

_No explicit latency/throughput targets, timeout values, or caching configuration are determinable from the code._

## Scalability

- **Statelessness:** The single resource class (`HelloResource`) holds no instance state and performs no database or external I/O, making the service inherently stateless and suitable for horizontal scaling.
- **Containerisation:** Four Docker build targets are provided (JVM, legacy-JAR, native, native-micro), all binding to `0.0.0.0:8080`, which is compatible with orchestrated horizontal scaling.
- **Replica counts / autoscaling:** _Not determinable from code._ No Kubernetes manifests, Helm charts, or autoscaling annotations are present in the snapshot.
- **Partitioning / sharding:** Not applicable; no data layer exists.

## Security

- **AuthN/AuthZ:** No authentication or authorisation mechanism is configured. The `/hello` endpoint is publicly accessible with no security middleware present.
- **Transport security (TLS):** Not configured. `application.properties` is empty; no `quarkus.http.ssl.*` properties are set. The container exposes plain HTTP on port 8080.
- **Secrets handling:** No secrets, credentials, or secret-management integrations (Vault, Kubernetes secrets, etc.) are referenced anywhere in the codebase.
- **Input validation:** The endpoint accepts no input parameters; no validation framework is applied.
- **Container user:** Containers run as non-root UID `185` (JVM images) or `1001` (native images), following least-privilege practice.
- **Security dependencies/middleware:** None detected beyond the Quarkus defaults.

## Observability

- **Logging:** The JBoss Log Manager (`org.jboss.logmanager.LogManager`) is configured as the JUL manager in both the Maven Surefire/Failsafe plugins and the Docker `JAVA_OPTS_APPEND`, enabling Quarkus structured logging to stdout. No custom log categories, levels, or appenders are defined in `application.properties`.
- **Metrics:** `quarkus-micrometer` or `quarkus-smallrye-metrics` are not included as dependencies; no metrics endpoint is exposed.
- **Tracing:** No distributed tracing extension (e.g., `quarkus-opentelemetry`) is present.
- **Health / readiness endpoints:** `quarkus-smallrye-health` is not listed as a dependency; no `/q/health` or `/q/health/ready` endpoints are configured.
- **Dev UI:** Available at `http://localhost:8080/q/dev/` in dev mode only (Quarkus built-in).

_Metrics, tracing, and health probes are not determinable from code beyond the absence noted above._

## Reliability

- **Resilience patterns:** No retry, circuit-breaker, bulkhead, or rate-limiting logic is present. The `quarkus-fault-tolerance` (SmallRye Fault Tolerance) extension is not included.
- **Timeouts:** No request or client timeout is configured.
- **Idempotency:** The sole endpoint (`GET /hello`) is read-only and side-effect-free, making it naturally idempotent.
- **Failure handling:** No explicit error handlers or exception mappers beyond Quarkus defaults are defined.
- **Availability / recovery:** No replica minimums, pod disruption budgets, or liveness probes are specified. Recovery behaviour relies entirely on the container orchestration layer, which is not defined in this repository.
