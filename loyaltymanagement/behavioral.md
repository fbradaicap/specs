---
repo: loyaltymanagement
spec_type: behavioral
commit: a0321cd4386bc946a394b585316e19a23f985828
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: 223517456ad62115d22dfd391894f67014ce1b9e9a6d6184885c09a215615c83
generated_at: 2026-06-30T17:38:11.525463360+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP (Spring Web MVC framework detected via `spring-boot-starter-webmvc`). An OpenAPI/Swagger UI is declared as a dependency (`springdoc-openapi-starter-webmvc-ui` v3.0.2), suggesting the intent to expose interactive API documentation at the standard SpringDoc path (`/swagger-ui.html` or `/swagger-ui/index.html`) and the OpenAPI spec at `/v3/api-docs`.

No controller classes, route mappings, or OpenAPI specification files are present in the provided source snapshot. No concrete endpoints can be documented.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| _Not determinable from code._ | | | | |

The H2 console is enabled (`spring-boot-h2console` dependency), typically accessible at `/h2-console` during development.

## Event Schemas

_Not determinable from code._

No messaging broker dependencies (Kafka, RabbitMQ, ActiveMQ, etc.), topic declarations, producer/consumer beans, or event payload classes are present in the snapshot.

## Input / Output Formats

- **Serialization:** JSON is the expected default serialization format, as implied by the Spring MVC framework (Jackson is bundled transitively with `spring-boot-starter-webmvc`).
- **Content types:** `application/json` is the standard default; actual `produces`/`consumes` annotations cannot be confirmed from the available source.
- **Pagination:** _Not determinable from code._
- **Request/response envelopes:** _Not determinable from code._

No DTO, model, or request/response wrapper classes are present in the snapshot to document field-level contracts.

## Error Handling

No `@ControllerAdvice`, `@ExceptionHandler`, `ProblemDetail`, or custom error response classes are present in the available source. Spring Boot's default error handling behaviour (auto-configured `BasicErrorController` returning a JSON error body with `timestamp`, `status`, `error`, `message`, and `path` fields) would apply by default, but no customisation is evidenced.

- **Default error format (Spring Boot auto-configuration):**
  ```json
  {
    "timestamp": "<ISO-8601>",
    "status": <HTTP status code>,
    "error": "<reason phrase>",
    "message": "<message>",
    "path": "<request path>"
  }
  ```
- **Custom exception mappings:** _Not determinable from code._
- **Validation behaviour:** _Not determinable from code._

## Versioning

No URI versioning prefix (e.g., `/v1/`, `/api/v2/`), `Accept`-header versioning, or schema evolution strategy is evidenced in the available source files. The artifact version is `0.0.1-SNAPSHOT`, indicating early/pre-release development stage.

_Not determinable from code._
