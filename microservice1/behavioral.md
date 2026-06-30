---
repo: microservice1
spec_type: behavioral
commit: 67a1a3cf92762515763f4e7d8cea0cfc4eeb30c1
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: 50ee4e55544c2ceaf0a54f62861f4c7d0de3a524a44ec92fd862370af5bb0606
generated_at: 2026-06-30T17:38:56.381083786+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP (Jakarta REST / JAX-RS via Quarkus `quarkus-rest`)  
**Base URL:** `http://<host>:8080`

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| `GET` | `/hello` | Returns a plain-text greeting string | No request body; no path/query parameters | `200 OK` — body: plain-text string (e.g. `"Hello"`) or JSON string (e.g. `"2"`) depending on content negotiation |

> **Note:** The `HelloResource` class declares two `@GET` methods on `@Path("/hello")` — one producing `text/plain` and one producing `application/json`. Both map to the same HTTP method and path; the active handler is selected by the `Accept` header sent by the client. The test suite asserts a `200` status with a plain-text body. The JSON-producing variant returns an integer serialised as a JSON string (`"2"`).

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Content types supported:**
  - `text/plain` — `GET /hello` returns the string `"Hello"` as plain text.
  - `application/json` — `GET /hello` returns the string `"2"` as a JSON string when the client signals `Accept: application/json`.
- **Serialization:** Plain string serialization for `text/plain`; standard JSON string encoding for `application/json` (no custom envelope or wrapper object is evidenced).
- **Pagination:** _Not determinable from code._ (no paginated endpoints exist)
- **Request envelope:** None — no request body is accepted by any endpoint.
- **Response envelope:** None — the response body is a bare scalar value (string) in both content-type variants.

---

## Error Handling

The source code contains no custom exception mappers, `ExceptionMapper` implementations, or explicit error-response DTOs. Quarkus / JAX-RS framework defaults therefore apply:

- `200 OK` is returned on successful invocation (confirmed by the test assertion `statusCode(200)`).
- Framework-level error codes (e.g. `404 Not Found` for unmatched paths, `406 Not Acceptable` for unsupported `Accept` headers, `500 Internal Server Error` for uncaught exceptions) are handled by the Quarkus runtime defaults.
- No custom validation logic or validation annotations (`@Valid`, `@NotNull`, etc.) are present in the resource class.
- No explicit error payload structure is defined in the service code; error response bodies for non-`2xx` responses are _Not determinable from code._

---

## Versioning

No URI versioning prefix (e.g. `/v1/`), no version request/response headers, and no schema-evolution strategy is present in the source code or configuration. _Not determinable from code._
