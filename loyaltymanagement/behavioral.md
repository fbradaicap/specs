---
repo: loyaltymanagement
spec_type: behavioral
commit: a0321cd4386bc946a394b585316e19a23f985828
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 469b02de1c019b90b692b2f0f3c1092f8f3229e17169b313f078fb1f6acdcbef
generated_at: 2026-06-30T15:59:26.665857914+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST (HTTP/JSON) via Spring Web MVC. The `springdoc-openapi-starter-webmvc-ui` dependency (v3.0.2) is present, indicating a Swagger UI/OpenAPI 3 specification is intended to be auto-generated and served at the standard SpringDoc path (`/swagger-ui.html` or `/swagger-ui/index.html`) and the OpenAPI JSON/YAML at `/v3/api-docs`.

No controller classes, request mappings, or route definitions are present in the provided source snapshot. Concrete endpoint contracts cannot be derived.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| _Not determinable from code._ | | | | |

The H2 console is also enabled as a dependency (`spring-boot-h2console`), suggesting a `/h2-console` management endpoint is available in development/test environments.

---

## Event Schemas

_Not determinable from code._

No messaging broker dependencies (Kafka, RabbitMQ, SQS, etc.), producer/consumer annotations, or topic/queue references are present in the snapshot.

---

## Input / Output Formats

- **Serialization:** JSON is the expected default content type, consistent with Spring Boot's auto-configured `Jackson` `MappingJackson2HttpMessageConverter` included transitively via `spring-boot-starter-webmvc`.
- **Content-Type header:** `application/json` (inferred from framework defaults).
- **OpenAPI UI:** The presence of `springdoc-openapi-starter-webmvc-ui:3.0.2` indicates the service is intended to self-document its contract via OpenAPI 3.0; the rendered specification would be the authoritative source for payload field shapes once controllers are implemented.
- **Pagination, envelopes, and field-level formats:** _Not determinable from code._ No DTOs, model classes, or repository projections are present in the snapshot.

---

## Error Handling

_Not determinable from code._

No `@ControllerAdvice`, `@ExceptionHandler`, `ResponseEntityExceptionHandler` subclasses, custom error DTOs, or validation constraint annotations (`@Valid`, `@NotNull`, etc.) are present in the provided source files. Spring Boot's default `/error` error-handling endpoint and `BasicErrorController` response structure (`timestamp`, `status`, `error`, `message`, `path`) will apply by default unless overridden.

---

## Versioning

_Not determinable from code._

No URI path versioning (e.g., `/v1/`, `/v2/`), `Accept`-header versioning, or API version properties are visible in the snapshot. The Maven artifact is versioned `0.0.1-SNAPSHOT`, indicating the service is in early/pre-release development. No explicit versioning strategy has been implemented in the available source.
