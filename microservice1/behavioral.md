---
repo: microservice1
spec_type: behavioral
commit: 67a1a3cf92762515763f4e7d8cea0cfc4eeb30c1
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d79e551fd821da0289be60837d42cd2e5e852eb45e459af0626dff12f227911c
generated_at: 2026-06-30T16:51:42.511630771+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST (HTTP) via Jakarta REST / Quarkus REST (`quarkus-rest`), served on port `8080`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| `GET` | `/hello` | Returns a greeting string (plain text) | No request body; no path/query parameters | `200 OK` — plain text body (e.g., `"Hello"`) |
| `GET` | `/hello` | Returns a numeric value as a string (JSON) | No request body; no path/query parameters | `200 OK` — JSON body (string representation of `2`) |

> **Note:** Both handlers are mapped to the same method and path (`GET /hello`) but are annotated with different `@Produces` media types (`text/plain` and `application/json` respectively). At runtime, content negotiation via the `Accept` request header determines which variant is invoked. The `HelloResourceTest` fixture asserts a `200` status with body `"Hello from Quarkus REST"` against `GET /hello`, though the source returns `"Hello"` — the exact response body in the deployed build is not fully determinable from the source alone.

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Content negotiation:** The `/hello` endpoint supports two `Content-Type` variants, selected by the client's `Accept` header:
  - `text/plain` — plain string response body.
  - `application/json` — JSON-encoded string response body.
- **Serialization:** Plain text and JSON (no structured object envelope; both responses are bare scalar strings).
- **Pagination:** Not applicable — responses are single scalar values.
- **Request body / parameters:** No request body, path parameters, or query parameters are defined on any endpoint.
- **Character encoding:** UTF-8 (configured via `project.build.sourceEncoding` and `project.reporting.outputEncoding` in `pom.xml`).

---

## Error Handling

No custom exception mappers, error DTOs, or explicit error-handling code are present in the source. Error behaviour therefore falls back to Quarkus / Jakarta REST framework defaults:

- `404 Not Found` — for unmatched paths (framework default).
- `405 Method Not Allowed` — for matched paths called with an unsupported HTTP method (framework default).
- `406 Not Acceptable` — if the client's `Accept` header cannot be satisfied (framework default content-negotiation behaviour).
- `500 Internal Server Error` — for unhandled runtime exceptions (framework default).

The structure of error response bodies for these cases is _Not determinable from code_ (depends on Quarkus default exception handling configuration, which is not overridden here).

---

## Versioning

No URI versioning prefix (e.g., `/v1/`), version request header, or schema-evolution strategy is present in the source. The Maven artifact is versioned `1.0.0-SNAPSHOT`, but this is not reflected in the API path or any HTTP header.

_Not determinable from code_ beyond the absence of any explicit versioning strategy.
