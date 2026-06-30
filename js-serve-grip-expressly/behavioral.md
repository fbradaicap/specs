---
repo: js-serve-grip-expressly
spec_type: behavioral
commit: 60e0350603e451b5ab32cc8883661b7b3e574c8c
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9232c1c7722e84c090d6d59cc77563f4961711f698c88abd545354e81f5d83f4
generated_at: 2026-06-30T14:52:34.254826147+02:00
generator: specsync
---

## API Contracts

This package is a **library/middleware**, not a standalone deployable microservice. It provides the `ServeGrip` middleware and associated adapter classes for integrating `@fanoutio/serve-grip` with the `@fastly/expressly` router framework on Fastly Compute@Edge. It exposes no HTTP endpoints of its own.

The routes shown in the README are **consumer-defined example routes** that a downstream application would register. The library makes the following patterns available to consumers:

**Protocol: HTTP (REST-style), running on Fastly Compute@Edge**

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| `GET` | `/api/stream` *(example)* | Open a long-lived GRIP HTTP stream on a channel | None | `text/plain`; `[stream open]\n` if proxied, `[not proxied]\n` otherwise |
| `GET` | `/api/publish` *(example)* | Publish a message to an HTTP stream channel | Query param: `msg` (string, optional) | `text/plain`; `Publish successful!` or error detail |
| `POST` | `/api/websocket` *(example)* | Handle WS-over-HTTP translated connection events | WebSocket-over-HTTP framing (GRIP) | Empty body; GRIP control headers set by middleware |
| `POST` | `/api/broadcast` *(example)* | Broadcast a message to a WebSocket channel | Raw text body | `text/plain`; `Ok\n` |

> All routes above are illustrative examples from the README. The library itself registers no endpoints.

---

## Event Schemas

_Not determinable from code._

No Kafka, RabbitMQ, or other message-broker event topics are produced or consumed by this library. The GRIP publish/subscribe mechanism operates over HTTP (GRIP protocol via `@fanoutio/grip` and `@fastly/grip-compute-js`) rather than a message broker.

---

## Input / Output Formats

- **Content type**: `text/plain` is used in all README examples for both requests and responses.
- **Serialization**: No JSON request/response envelopes are defined by the library itself. The publish payload for WS-over-HTTP broadcast is raw text passed directly to `WebSocketMessageFormat`.
- **GRIP control**: GRIP instructions (`Grip-Hold`, `Grip-Channel`, etc.) are communicated via HTTP response headers, set by `@fanoutio/serve-grip` through the `gripInstruct` object exposed on `res.grip`.
- **Request body access**: `ExpresslyApiRequest.getBody()` reads the full request body as a `Buffer` via `arrayBuffer()`. No streaming or chunked handling is defined at the library level.
- **Pagination**: _Not determinable from code._
- **Request envelope**: No envelope; the library patches `req.grip` and `res.grip` properties onto the Expressly `ERequest`/`EResponse` objects via middleware.

---

## Error Handling

Evidence from the README examples and source:

| Scenario | Status Code | Response Body |
|----------|-------------|---------------|
| Publish failure (exception thrown by publisher) | `500` | `text/plain`: `Publish failed!\n<message>\n<JSON context>` |
| Non-WebSocket request to WS-over-HTTP endpoint | `400` | `text/plain`: `[not a websocket request]\n` |

- Error detail is extracted from the caught exception as `{ message, context }` and the `context` field is serialised with `JSON.stringify(context, null, 2)`.
- GRIP signature validation and connection-ID checking are delegated to `@fanoutio/serve-grip`; the specific error responses for invalid GRIP signatures are _not determinable from this library's code_ (they are handled upstream in the `serve-grip` dependency).
- `ExpresslyApiResponse.setStatus(value: number)` sets `this._res.status` directly, providing a generic mechanism for the middleware layer to apply arbitrary HTTP status codes.

---

## Versioning

- The library is published as `@fastly/serve-grip-expressly` at version **`0.1.1`** (semver, recorded in `package.json`).
- No URI-based, header-based, or schema-evolution versioning strategy is implemented within the library itself.
- The peer dependency on `@fastly/expressly` is pinned to `1.0.0-alpha.1`, indicating the API is considered pre-stable.
- No API version prefix (e.g., `/v1/`) appears in any endpoint pattern evidenced in the code.
