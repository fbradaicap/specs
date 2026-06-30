---
repo: microservice1
spec_type: behavioral
commit: 954d85981a924722137ae7edea41a6ffbf5b2444
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d7c1d6d7e1a8acbea7e48f4ff300b586a4fb468527117a2559d0a34869f35a63
generated_at: 2026-06-30T16:00:13.254039078+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST (HTTP) via Jakarta REST (JAX-RS), implemented with Quarkus REST (`io.quarkus:quarkus-rest`). The service listens on port `8080`.

| Method | Path     | Purpose                        | Request                  | Response                                      |
|--------|----------|--------------------------------|--------------------------|-----------------------------------------------|
| `GET`  | `/hello` | Returns a plain-text greeting  | No request body or parameters | `200 OK` — plain-text string (e.g. `"Hello"`) |

> **Note:** The source class `HelloResource.java` returns the literal `"Hello"`, while the test asserts the body is `"Hello from Quarkus REST"`. The exact runtime response body may differ; the response type is `text/plain` in both cases.

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Content type (response):** `text/plain` (`jakarta.ws.rs.core.MediaType.TEXT_PLAIN`), as declared by `@Produces(MediaType.TEXT_PLAIN)`.
- **Serialization:** Plain string — no JSON, Protobuf, or Avro serialization is used.
- **Request body:** None — the single endpoint accepts no request body or query parameters.
- **Pagination:** Not applicable — the endpoint returns a single scalar value.
- **Request/response envelope:** None — the response body is a raw string with no wrapper structure.

---

## Error Handling

No custom error handling, exception mappers, or validation logic is evidenced in the source code. The following behaviour is implied by the Quarkus/JAX-RS defaults:

| Condition                          | Expected Status Code | Notes                                         |
|------------------------------------|----------------------|-----------------------------------------------|
| Successful `GET /hello`            | `200 OK`             | Confirmed by `HelloResourceTest`              |
| Method not allowed (e.g. `POST`)   | `405 Method Not Allowed` | JAX-RS framework default                  |
| Path not found                     | `404 Not Found`      | JAX-RS framework default                      |

No custom error payload structure is defined in the codebase.

---

## Versioning

No API versioning strategy (URI prefix, `Accept` header versioning, or schema evolution mechanism) is evident in the source code. The single endpoint is exposed directly at `/hello` with no version segment.
