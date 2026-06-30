---
repo: expressly
spec_type: behavioral
commit: d15cb937392edb46d2e39d406240dc6fdbf78c6b
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4e4d8e267b36915bdf0c4034e74b77d1ec77facdcabde45d5d705636dbf3fd75
generated_at: 2026-06-30T14:53:20.999053512+02:00
generator: specsync
---

## API Contracts

**expressly** is a library/framework (not a deployed service with a fixed HTTP surface). It provides an Express-style routing API for **Fastly Compute** edge functions. The "endpoints" extracted are artefacts of its internal CORS preflight handling logic, not standalone service routes.

**Protocol:** HTTP (all standard methods supported via the router's method helpers, dispatched by the Fastly Compute `fetch` event listener).

### Router Method API

The `Router` class exposes the following route-registration methods. Each accepts a URL pattern (using `path-to-regexp` syntax) and an async `(req, res) => Promise<any>` callback.

| Method helper | HTTP Method(s) matched | Pattern type |
|---|---|---|
| `router.get(pattern, cb)` | `GET` | `path-to-regexp` string |
| `router.post(pattern, cb)` | `POST` | `path-to-regexp` string |
| `router.put(pattern, cb)` | `PUT` | `path-to-regexp` string |
| `router.delete(pattern, cb)` | `DELETE` | `path-to-regexp` string |
| `router.head(pattern, cb)` | `HEAD` | `path-to-regexp` string |
| `router.options(pattern, cb)` | `OPTIONS` | `path-to-regexp` string |
| `router.patch(pattern, cb)` | `PATCH` | `path-to-regexp` string |
| `router.purge(pattern, cb)` | `PURGE` | `path-to-regexp` string |
| `router.all(pattern, cb)` | All methods (`*`) | `path-to-regexp` string |
| `router.route(methods[], pattern, cb)` | Arbitrary method list | `path-to-regexp` string |
| `router.use([path,] cb)` | All methods (middleware) | Optional path prefix |

### Built-in CORS Preflight Handler

When `config.autoCorsPreflight` is enabled, the router automatically registers an `OPTIONS *` handler with the following behaviour:

| Method | Path | Purpose | Request | Response |
|---|---|---|---|---|
| `OPTIONS` | `*` | Automatic CORS preflight | `origin`, `access-control-request-method`, `access-control-request-headers` headers | `200` with `access-control-allow-origin`, `access-control-allow-methods`, `access-control-allow-headers`; or `403` if origin not trusted |

### Router Constructor Configuration (`EConfig`)

| Field | Type | Default | Description |
|---|---|---|---|
| `parseCookie` | `boolean` | `true` | Parse `Cookie` header |
| `auto405` | `boolean` | `true` | Return `405` on method mismatch (vs. `404`) |
| `extractRequestParameters` | `boolean` | `true` | Extract path params from URL |
| `autoContentType` | `boolean` | `false` | Automatically set `Content-Type` |
| `autoCorsPreflight` | `{ trustedOrigins: string[] }` | `undefined` | Enable automatic CORS preflight handling |

---

## Event Schemas

_Not determinable from code._

No message broker, Kafka topics, RabbitMQ queues, or other async event channels are present. The `mitt` dependency is used for internal in-process event emission (e.g., the `finish` event on `ERes` carries a `Response | undefined` payload), not for external messaging.

---

## Input / Output Formats

- **Serialization:** JSON is the primary output format. The response object exposes a `.json(body)` method that serializes a JavaScript object to JSON and sets the response body accordingly.
- **Content-Type:** Automatic `Content-Type` header setting is opt-in via `autoContentType: true` in `EConfig`. When disabled (default), callers must set it manually.
- **Cookie parsing:** Incoming `Cookie` headers are parsed using the `cookie` package when `parseCookie: true` (default). The `Set-Cookie` response header is managed by the response abstraction.
- **Path parameters:** URL path parameters extracted via `path-to-regexp` are available on the request object when `extractRequestParameters: true` (default).
- **Request/Response types:** `EReq` and `ERes` are the base types; both are generic and extensible. `EResponse` wraps the native Fastly Compute `Response` and is serialized to a native `Response` at the end of the handler chain via `Router.serializeResponse(res)`.
- **Pagination:** _Not determinable from code._
- **Request envelope:** No wrapper envelope; raw HTTP request/response model following Express conventions.

---

## Error Handling

The default error handler (`defaultErrorHandler`) maps exceptions to HTTP responses as follows:

| Exception type | Condition | HTTP Status | Response body |
|---|---|---|---|
| `ErrorNotFound` | Route not matched | `404` | _Not determinable from code (sendStatus only)_ |
| `ErrorMethodNotAllowed` | Method mismatch, `auto405: false` | `404` | _Not determinable from code (sendStatus only)_ |
| `ErrorMethodNotAllowed` | Method mismatch, `auto405: true` | `405` | Sets `Allow` header to permitted methods; no body evidenced |
| Any other `Error` | Unhandled exception | `500` | `{ "error": "<err.message>" }` (JSON) |

- **Auto-405 behaviour:** When `auto405: true` (default), a request whose path matches a registered route but whose HTTP method does not will receive a `405 Method Not Allowed` response with the `Allow` header populated with the set of methods that do match.
- **Error middleware:** Custom error middleware can be registered via `router.use((err, req, res) => …)` (three-argument callback). The router detects error handlers by checking `cb.length === 3` and routes thrown exceptions through the error handler stack.
- **Validation:** _Not determinable from code._
- **Error payload structure (500):** `{ "error": string }` — JSON object with a single `error` field containing the exception message.

---

## Versioning

The library is published as `@fastly/expressly` at version **2.4.0** (semver). No URI-based versioning, header-based versioning, or schema evolution strategy is applied to the routing API itself — versioning is managed exclusively through npm package versioning. No API version prefix (e.g., `/v1/`) is enforced or evidenced in the routing layer.
