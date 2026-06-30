---
repo: keycloak-kafka
spec_type: behavioral
commit: e9c3f89d676b7047cdf8d6c9ddbd6805d15386c6
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4b28decdb78ed7c8135d83c1e3788ad4325616162fec5e4eb63ea6ceca528481
generated_at: 2026-06-30T14:53:59.375613323+02:00
generator: specsync
---

## API Contracts

This service exposes no HTTP/REST, GraphQL, or gRPC API. It is a **Keycloak SPI (Service Provider Interface) extension** — specifically an `EventListenerProvider` — that integrates directly into the Keycloak server process and communicates outbound via Apache Kafka.

The "endpoints" extracted (`topicEvents`, `clientId`, `bootstrapServers`, `topicAdminEvents`, `events`) are **SPI configuration parameters**, not HTTP endpoints. They are supplied either via Keycloak start-command flags or environment variables:

| SPI Config Key | Env Variable | Required | Purpose |
|---|---|---|---|
| `topicEvents` | `KAFKA_TOPIC` | Yes | Kafka topic for user events |
| `clientId` | `KAFKA_CLIENT_ID` | Yes | Kafka producer `client.id` |
| `bootstrapServers` | `KAFKA_BOOTSTRAP_SERVERS` | Yes | Comma-separated Kafka broker list |
| `events` | `KAFKA_EVENTS` | No (defaults to `REGISTER`) | Comma-separated Keycloak event types to forward |
| `topicAdminEvents` | `KAFKA_ADMIN_TOPIC` | No | Kafka topic for admin events; disabled if unset |

The Keycloak SPI factory ID (registered provider name) is `kafka`.

## Event Schemas

**Broker:** Apache Kafka (tested with versions `2.12-2.1.x` through `2.13-3.3.x`)

### Produced Topics

#### Topic: `topicEvents` (value of `KAFKA_TOPIC`, e.g. `keycloak-events`)

- **Trigger:** A Keycloak user-facing event whose `EventType` matches one of the configured `events` values.
- **Payload:** JSON serialization of Keycloak's `org.keycloak.events.Event` object, produced via Jackson `ObjectMapper`.
- **Key:** `null` (no record key is set; `ProducerRecord` is constructed with topic + value only).
- **Value serializer:** `org.apache.kafka.common.serialization.StringSerializer` (UTF-8 string).
- The fields of the `Event` payload are determined by Keycloak's core `Event` model; the exact field set is _Not determinable from code_ (defined in `keycloak-core`), but includes at minimum `type` (an `EventType` enum value such as `REGISTER`, `LOGIN`, `CLIENT_DELETE`, etc.).

#### Topic: `topicAdminEvents` (value of `KAFKA_ADMIN_TOPIC`)

- **Trigger:** Any Keycloak `AdminEvent`, when `topicAdminEvents` is non-null.
- **Payload:** JSON serialization of Keycloak's `org.keycloak.events.admin.AdminEvent` object, produced via Jackson `ObjectMapper`.
- **Key:** `null`.
- **Value serializer:** `org.apache.kafka.common.serialization.StringSerializer`.
- The exact field set of `AdminEvent` is _Not determinable from code_ (defined in `keycloak-core`).

### Consumed Topics

None. This module is a producer only; it does not consume any Kafka topics.

## Input / Output Formats

- **Serialization format:** JSON, produced using Jackson `ObjectMapper#writeValueAsString()`. No custom serializers or schema registry (e.g., Avro/Schema Registry) are used.
- **Kafka record key:** Always `null` (no partitioning key set).
- **Kafka record value:** UTF-8 JSON string (via `StringSerializer` for both key and value).
- **Send behaviour:** Synchronous from the caller's perspective — `producer.send()` returns a `Future<RecordMetadata>`, which is resolved with a 30-second timeout (`metaData.get(30, TimeUnit.SECONDS)`).
- There is no HTTP request/response envelope, pagination, or content-type negotiation — the module has no HTTP interface.

## Error Handling

| Situation | Behaviour |
|---|---|
| Unknown/invalid `EventType` string in `events` config | Logged at `DEBUG` level and silently ignored; the invalid type is excluded from the active filter list. |
| `topicEvents` is `null` at init | Throws `NullPointerException` with message `"topic must not be null."`, preventing Keycloak from starting. |
| `clientId` is `null` at init | Throws `NullPointerException` with message `"clientId must not be null."`, preventing Keycloak from starting. |
| `bootstrapServers` is `null` at init | Throws `NullPointerException` with message `"bootstrapServers must not be null"`, preventing Keycloak from starting. |
| `events` is `null` or empty at init | Defaults silently to `["REGISTER"]`. |
| `topicAdminEvents` is `null` | Admin events are silently suppressed (no Kafka message produced). |
| `JsonProcessingException`, `ExecutionException`, `TimeoutException` during produce | Logged at `ERROR` level via JBoss Logging; exception is swallowed (event is lost). |
| `InterruptedException` during produce | Logged at `ERROR` level; `Thread.currentThread().interrupt()` is called to restore interrupt status; event is lost. |

There is no retry logic beyond the Kafka producer's own built-in retry configuration (controllable via the `retries` producer property).

## Versioning

- The module itself is versioned as `1.1.3` (Maven artifact `com.github.snuk87.keycloak:keycloak-kafka:1.1.3`).
- There is no API versioning strategy (no URI versioning, no version headers) because the module exposes no HTTP interface.
- The Keycloak SPI registration ID is the fixed string `"kafka"` — there is no version qualifier in this identifier.
- Compatibility with specific Keycloak and Kafka versions is documented in the README but there is no schema evolution mechanism in the code; payload schema is implicitly tied to the Keycloak `Event`/`AdminEvent` model version at build time.
