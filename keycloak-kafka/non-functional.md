---
repo: keycloak-kafka
spec_type: non_functional
commit: e9c3f89d676b7047cdf8d6c9ddbd6805d15386c6
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4b28decdb78ed7c8135d83c1e3788ad4325616162fec5e4eb63ea6ceca528481
generated_at: 2026-06-30T14:53:59.375613323+02:00
generator: specsync
---

## Performance

- **Producer send timeout:** Each Kafka `producer.send()` call is blocked synchronously with a hard timeout of **30 seconds** (`metaData.get(30, TimeUnit.SECONDS)`). If the broker does not acknowledge within this window, a `TimeoutException` is logged and the event is dropped.
- **`max.block.ms`:** Configurable via `--spi-events-listener-kafka-max-block-ms` (e.g., 10 000 ms per README example); controls how long the producer will block when the send buffer is full. No default override is applied in code — the Kafka client default (60 000 ms) applies unless explicitly set.
- **Kafka producer tuning:** The full set of Kafka producer properties (`batch.size`, `linger.ms`, `buffer.memory`, `compression.type`, `acks`, `delivery.timeout.ms`, `request.timeout.ms`, etc.) is passable at startup via SPI config parameters; none are overridden from their Kafka-client defaults in code.
- **Serialisation:** Events are serialised to JSON via Jackson `ObjectMapper` inline on the Keycloak event-listener thread, adding serialisation latency to every Keycloak authentication event.
- **Singleton producer instance:** A single `KafkaProducer` instance is shared across all Keycloak sessions (created once in the factory's `create()` method). This avoids per-request connection overhead but means all events share one producer's internal buffer and network thread.
- **Caching:** No application-level caching is present. _Not determinable from code_ whether the Keycloak host applies any additional caching layer.
- **docker-compose topic configuration:** `KAFKA_CREATE_TOPICS: "keycloak-events:1:1"` — 1 partition, replication factor 1 (sample/dev only).

---

## Scalability

- **Stateless producer logic:** The `KafkaEventListenerProvider` itself holds no user-session state; however, the factory creates only a single shared instance (`if (instance == null)`), making the provider a **per-JVM singleton**. Horizontal scaling of Keycloak nodes produces independent producer instances on each node.
- **Replica count:** The docker-compose sample deploys **one** Keycloak container, **one** Kafka broker, and **one** Zookeeper node. No autoscaling configuration is defined.
- **Partitioning:** The Kafka topic is created with **1 partition** in the sample deployment. The `partitioner.class` producer property is configurable but no custom partitioner is implemented.
- **Kafka client scalability:** Because the module uses a standard Kafka producer, it inherits Kafka's producer-side scalability characteristics. Multiple Keycloak nodes can write to the same topic concurrently without coordination.
- **Vertical scaling:** No JVM heap, CPU, or memory resource limits are specified in any deployment manifest.
- **Autoscaling:** _Not determinable from code._

---

## Security

- **Transport security (Kafka):** SSL/TLS to the Kafka broker is **supported but not enabled by default**. The following properties are explicitly enumerated and passable at startup: `security.protocol`, `ssl.keystore.location`, `ssl.keystore.password`, `ssl.key.password`, `ssl.truststore.location`, `ssl.truststore.password`, `ssl.truststore.type`, `ssl.keystore.type`, `ssl.enabled.protocols`, `ssl.protocol`, `ssl.cipher.suites`, `ssl.endpoint.identification.algorithm`, and related SSL/TLS tuning parameters. The docker-compose sample uses `PLAINTEXT` (unauthenticated, unencrypted).
- **SASL authentication:** SASL mechanisms (PLAIN, GSSAPI/Kerberos, OAUTHBEARER) are supported via configurable properties: `sasl.mechanism`, `sasl.jaas.config`, `sasl.kerberos.service.name`, and related SASL refresh/login parameters.
- **Secrets handling:** Kafka credentials (`ssl.keystore.password`, `ssl.truststore.password`, `ssl.key.password`, `sasl.jaas.config`) are passed as plaintext startup parameters or environment variables. No secrets-manager integration or secret masking is evident in the code.
- **AuthN/AuthZ (Keycloak HTTP):** The module does not expose its own HTTP endpoints. Authentication and authorisation for the Keycloak admin and user-facing endpoints are handled entirely by the Keycloak runtime. The docker-compose sample sets `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` as plaintext environment variables (`admin`/`admin`).
- **Input validation:** Event-type strings supplied via `KAFKA_EVENTS` / `--spi-events-listener-kafka-events` are validated against the `EventType` enum; unrecognised values are silently ignored. Required configuration values (`topicEvents`, `clientId`, `bootstrapServers`) are validated for non-null at startup; a `NullPointerException` is thrown and startup is aborted if they are absent.
- **Transport security (Keycloak HTTP):** The Dockerfile and docker-compose use `start-dev` / the base Keycloak image without explicit TLS configuration. Production TLS termination is not configured in the provided manifests.
- **Notable security dependency:** `keycloak-server-spi` / `keycloak-server-spi-private` v21.0.1 provide the security framework; the module relies entirely on Keycloak's own AuthN/AuthZ infrastructure.

---

## Observability

- **Logging:** Uses `org.jboss.logging.Logger` throughout. Log statements include:
  - `INFO`: Module initialisation (`"Init kafka module ..."`).
  - `DEBUG`: Per-event produce attempts and confirmations (`"Produce to topic: ... ..."`, `"Produced to topic: ..."`).
  - `ERROR`: All caught exceptions during event production (`JsonProcessingException`, `ExecutionException`, `TimeoutException`, `InterruptedException`).
  - `DEBUG`: Unknown/invalid event-type strings at startup.
- **Metrics:** No application-level metrics instrumentation (e.g., Micrometer, Prometheus) is present in the module code. Kafka producer built-in JMX metrics are available if the JMX agent is configured on the host JVM. Keycloak's own metrics endpoint (if enabled) is not extended by this module.
- **Tracing:** _Not determinable from code._ No distributed tracing integration (e.g., OpenTelemetry, Jaeger) is implemented.
- **Health/readiness endpoints:** No custom health or readiness probes are implemented in this module. Any such endpoints are provided by the Keycloak runtime itself. No `livenessProbe` or `readinessProbe` is configured in the deployment manifests.
- **Kafka producer metrics:** The `metric.reporters`, `metrics.num.samples`, `metrics.recording.level`, and `metrics.sample.window.ms` producer properties are configurable, allowing external metrics reporters to be plugged in (target), but none are configured by default.

---

## Reliability

- **Synchronous send with timeout:** `producer.send(record).get(30, TimeUnit.SECONDS)` blocks the Keycloak event-listener thread. On timeout or broker unavailability, the event is **permanently lost** (logged at ERROR, not retried or queued).
- **No retry logic in application code:** Retries are not implemented at the application level. The Kafka producer's built-in `retries` property is configurable but not set to a non-default value in code. `retry.backoff.ms` and `reconnect.backoff.ms`/`reconnect.backoff.max.ms` are also configurable but unset.
- **No circuit breaker:** No circuit-breaker pattern (e.g., Resilience4j, Hystrix) is present. A sustained Kafka broker outage will cause every affected Keycloak event to block for up to 30 seconds before failing.
- **Idempotence:** The Kafka producer `enable.idempotence` property is configurable but not enabled by default. Duplicate events on producer retry are therefore possible if retries are configured.
- **Interrupt handling:** `InterruptedException` is caught and `Thread.currentThread().interrupt()` is re-set, following best practice to preserve the interrupted status of the Keycloak worker thread.
- **Singleton instance risk:** The shared `KafkaEventListenerProvider` instance (see Performance) is not thread-safe by explicit design; concurrent Keycloak threads sharing the single `ObjectMapper` and `Producer` rely on those classes being thread-safe (both Jackson's `ObjectMapper` and `KafkaProducer` are documented as thread-safe).
- **Fail-fast startup:** Missing required configuration (`topicEvents`, `clientId`, `bootstrapServers`) causes immediate startup failure, preventing silent misconfiguration.
- **Single-node Kafka (sample):** The docker-compose deployment has replication factor 1 and no broker redundancy. This configuration offers no fault tolerance for the broker and is unsuitable for production availability requirements.
- **No dead-letter queue or event persistence:** Events that fail to produce are logged and discarded. There is no fallback store, dead-letter topic, or replay mechanism.
