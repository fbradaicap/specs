---
repo: keycloak-kafka
spec_type: technical
commit: e9c3f89d676b7047cdf8d6c9ddbd6805d15386c6
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4b28decdb78ed7c8135d83c1e3788ad4325616162fec5e4eb63ea6ceca528481
generated_at: 2026-06-30T14:53:59.375613323+02:00
generator: specsync
---

## Tech Stack

- **Language:** Java 17 (`maven.compiler.source` / `maven.compiler.target` = 17)
- **Build tool:** Apache Maven (with `maven-assembly-plugin` for fat-jar packaging)
- **Runtime framework:** Keycloak 21.0.1 SPI (Service Provider Interface) — the module runs embedded inside a Keycloak server instance
- **Notable runtime libraries:**
  - `org.apache.kafka:kafka-clients:3.3.1` — Kafka producer client
  - `org.jboss.logging:jboss-logging:3.3.2.Final` — logging facade
  - `com.fasterxml.jackson.databind:ObjectMapper` — JSON serialisation of events (pulled transitively via Keycloak dependencies)
- **Test libraries:** JUnit Jupiter 5.8.1 (`junit-jupiter-api`)

---

## Architecture Patterns

- **Plugin / extension pattern:** The service is not a standalone process; it is a Keycloak EventListener SPI module deployed as a fat JAR into `$KEYCLOAK_HOME/providers`. Keycloak loads it automatically on startup.
- **Event-driven (outbound producer):** The module reacts to Keycloak lifecycle events and admin events and produces them asynchronously to Kafka topics. There is no inbound message consumption.
- **Factory pattern:** A `KafkaProducerFactory` interface decouples producer construction from business logic, enabling `KafkaMockProducerFactory` substitution in tests.
- **Singleton provider:** `KafkaEventListenerProviderFactory` lazily creates a single `KafkaEventListenerProvider` instance and reuses it across all Keycloak sessions.
- **Key internal components:**
  - `KafkaEventListenerProviderFactory` — SPI factory; reads configuration, initialises the Kafka producer settings, and acts as the entry point registered under the SPI id `"kafka"`.
  - `KafkaEventListenerProvider` — implements `EventListenerProvider`; routes `onEvent(Event)` and `onEvent(AdminEvent, …)` calls to the appropriate Kafka topic.
  - `KafkaProducerConfig` / `KafkaProducerProperty` — exhaustive enumeration of supported Kafka producer properties read from Keycloak SPI `Scope` config.
  - `KafkaStandardProducerFactory` / `KafkaProducerFactory` — creates a `KafkaProducer<String, String>` with string serialisers.

---

## Database & Data Ownership

This service owns **no database or persistent datastore**. It acts purely as a Keycloak SPI event bridge; all identity and session data is owned by Keycloak itself. No migrations, schema definitions, or ORM entities are present.

---

## Dependencies

### Runtime dependencies

| Dependency | Version | Role |
|---|---|---|
| `org.apache.kafka:kafka-clients` | 3.3.1 | Kafka `KafkaProducer` for publishing events |
| `org.jboss.logging:jboss-logging` | 3.3.2.Final | Logging facade (bundled in fat JAR) |
| Apache Kafka broker | 2.12-2.1.x – 2.13-3.3.x (tested) | External message broker; address configured via `KAFKA_BOOTSTRAP_SERVERS` |
| Keycloak server | 21.0.1 (target) | Host runtime; provides SPI classes (`keycloak-core`, `keycloak-server-spi`, `keycloak-server-spi-private`) — all `provided` scope, not bundled |

### Build-only dependencies

| Dependency | Version | Role |
|---|---|---|
| `maven-assembly-plugin` | (Maven default) | Produces `jar-with-dependencies` fat JAR |
| `org.junit.jupiter:junit-jupiter-api` | 5.8.1 | Unit testing (`test` scope) |

### Infrastructure dependencies (docker-compose)

| Service | Image | Role |
|---|---|---|
| Apache ZooKeeper | `bitnami/zookeeper:latest` | Kafka coordination (dev/sample environment) |
| Apache Kafka | `bitnami/kafka:latest` | Message broker (dev/sample environment) |

---

## Deployment Model

### Container image
- **Base image:** `quay.io/keycloak/keycloak:19.0.3` (note: Dockerfile targets Keycloak 19 while `pom.xml` targets 21.0.1; the Dockerfile downloads the pre-built 1.1.1 release JAR rather than the current build).
- The fat JAR is copied into `/opt/keycloak/providers/`; Keycloak discovers and installs it automatically at startup.
- `RUN /opt/keycloak/bin/kc.sh build` bakes the provider into the optimised Keycloak image layer.
- **Entrypoint:** `/opt/keycloak/bin/kc.sh`

### Orchestration
- **Docker Compose** (`docker-compose.yml`) defines a three-service local stack: `keycloak`, `zookeeper`, and `kafka`.
- Keycloak is exposed on **port 8080** (host → container).
- Kafka inter-service communication uses port **9094** (OUTSIDE listener); external access via **9092**.

### Configuration (environment variables)

| Variable | SPI parameter | Description | Required |
|---|---|---|---|
| `KAFKA_TOPIC` | `--spi-events-listener-kafka-topic-events` | Kafka topic for user events | Yes |
| `KAFKA_CLIENT_ID` | `--spi-events-listener-kafka-client-id` | Kafka producer client ID | Yes |
| `KAFKA_BOOTSTRAP_SERVERS` | `--spi-events-listener-kafka-bootstrap-servers` | Comma-separated broker list | Yes |
| `KAFKA_EVENTS` | `--spi-events-listener-kafka-events` | Comma-separated Keycloak event types to forward (default: `REGISTER`) | No |
| `KAFKA_ADMIN_TOPIC` | `--spi-events-listener-kafka-topic-admin-events` | Kafka topic for admin events | No |
| `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | — | Keycloak bootstrap admin credentials | Yes (Keycloak) |

Additional Kafka producer properties (SSL/TLS, SASL, timeouts, etc.) can be passed via `--spi-events-listener-kafka-<property-name>` parameters at Keycloak startup.

### Health / readiness endpoints
_Not determinable from code._ Health checks are inherited from the base Keycloak image; no custom endpoints are defined in this module.
