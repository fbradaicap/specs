---
repo: keycloak-kafka
spec_type: functional
commit: e9c3f89d676b7047cdf8d6c9ddbd6805d15386c6
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4b28decdb78ed7c8135d83c1e3788ad4325616162fec5e4eb63ea6ceca528481
generated_at: 2026-06-30T14:53:59.375613323+02:00
generator: specsync
---

## Business Purpose

This service is a Keycloak SPI (Service Provider Interface) extension module that bridges Keycloak's internal event system to Apache Kafka. It captures authentication and administrative events emitted by a Keycloak identity server and publishes them as JSON messages to configured Kafka topics, enabling downstream systems to react to identity lifecycle events (e.g., user registration, login, admin actions) in an asynchronous, decoupled manner.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Identity Event Streaming — the narrow concern of forwarding Keycloak-generated events to a message broker.
- **Core domain entities / aggregates:**
  - `Event` (Keycloak user/authentication event, e.g. `REGISTER`, `LOGIN`)
  - `AdminEvent` (Keycloak admin-plane event, e.g. realm/client/user management actions)
- **Relationships to neighbouring contexts:**
  - **Upstream:** Keycloak server (provides `Event` and `AdminEvent` via the `EventListenerProvider` SPI).
  - **Downstream:** Any Kafka consumer subscribed to the configured topics (`topicEvents`, `topicAdminEvents`); this module has no knowledge of those consumers.

## Use Cases / User Stories

- **As a platform operator**, I want Keycloak user-facing events (e.g., REGISTER, LOGIN) to be published to a Kafka topic so that downstream services can react to authentication lifecycle changes in real time. _(→ `onEvent(Event)` → produces to `topicEvents`)_
- **As a platform operator**, I want Keycloak admin events (e.g., user/realm/client management) to be published to a separate Kafka topic so that audit and provisioning systems can consume them independently. _(→ `onEvent(AdminEvent, boolean)` → produces to `topicAdminEvents`)_
- **As a platform operator**, I want to filter which event types are forwarded to Kafka so that only relevant events consume broker resources. _(→ `events` / `KAFKA_EVENTS` configuration)_
- **As a platform operator**, I want to configure the Kafka producer (bootstrap servers, client ID, SSL/TLS, SASL, timeouts, etc.) via environment variables or Keycloak SPI startup parameters so that the module can connect to secured or custom broker setups without code changes. _(→ `KafkaProducerConfig`, `KafkaEventListenerProviderFactory.init()`)_
- **As a developer**, I want to deploy the module as a single fat JAR into `$KEYCLOAK_HOME/providers` so that no manual classpath management is required.

## Business Rules

- `topicEvents` (`KAFKA_TOPIC`) **must** be configured; the module throws `NullPointerException` on startup if absent.
- `clientId` (`KAFKA_CLIENT_ID`) **must** be configured; the module throws `NullPointerException` on startup if absent.
- `bootstrapServers` (`KAFKA_BOOTSTRAP_SERVERS`) **must** be configured; the module throws `NullPointerException` on startup if absent.
- If `events` (`KAFKA_EVENTS`) is not configured or resolves to an empty list, the module defaults to forwarding only the `REGISTER` event type.
- User-facing events are only forwarded to Kafka if the event's `EventType` exactly matches one of the configured (whitelist) event types; unrecognised type strings in configuration are silently ignored.
- Admin events are only forwarded if `topicAdminEvents` (`KAFKA_ADMIN_TOPIC`) is explicitly set; if this property is absent/null, all admin events are silently dropped.
- Events are serialised as JSON (via Jackson `ObjectMapper`) before being produced to Kafka; the message key is `null` (no partitioning key set).
- The Kafka producer `send()` call blocks for a maximum of **30 seconds**; if the send does not complete within this timeout, the event is dropped and an error is logged.
- The `KafkaEventListenerProvider` instance is a **singleton** per factory lifecycle (lazy-initialised on first `create()` call); all Keycloak sessions share the same producer instance. _(inferred)_
- Event type matching is case-insensitive during configuration parsing (`toUpperCase()` applied to configured event name strings).
- The `includeRepresentation` flag passed to `onEvent(AdminEvent, boolean)` is accepted but not used — admin event representation is always serialised in full. _(inferred)_
