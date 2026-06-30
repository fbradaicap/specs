---
repo: expressly
spec_type: functional
commit: d15cb937392edb46d2e39d406240dc6fdbf78c6b
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4e4d8e267b36915bdf0c4034e74b77d1ec77facdcabde45d5d705636dbf3fd75
generated_at: 2026-06-30T14:53:20.999053512+02:00
generator: specsync
---

## Business Purpose

`@fastly/expressly` is a TypeScript routing library for **Fastly Compute** (edge workers) that provides an Express-style request/response API. It allows developers to define HTTP route handlers and middleware at the edge, enabling lightweight application logic — including CORS preflight handling, cookie parsing, and JSON responses — to run inside Fastly's Compute environment without a traditional server.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge HTTP routing / middleware framework for Fastly Compute.
- **Core entities / aggregates:**
  - `Router` — the central aggregate; owns the ordered handler and error-handler stacks, routing configuration, and the top-level `fetch` event listener.
  - `ERequest` (`EReq`) — edge-adapted HTTP request abstraction (wraps the Compute `FetchEvent`).
  - `EResponse` (`ERes`) — mutable HTTP response builder.
  - `RequestHandler` — pairs a route-match function with a handler callback.
  - `ErrorMiddleware` — pairs a route-match function with an error-handler callback.
- **Neighbouring contexts:**
  - **Upstream:** Fastly Compute runtime (`@fastly/js-compute`) — provides the `FetchEvent` and native `Response` primitives that `Router` consumes.
  - **Downstream:** Application code written by end-users — extends `Router`, `EReq`, and `ERes` with custom generic types and business logic.

## Use Cases / User Stories

- **As an edge application developer**, I want to register route handlers by HTTP method and URL pattern (`router.get(pattern, cb)`, `router.post(pattern, cb)`, etc.) so that different business logic runs for different request paths.
- **As an edge application developer**, I want to register catch-all or path-scoped middleware (`router.use(...)`) so that cross-cutting concerns (logging, authentication) execute before/after route handlers.
- **As an edge application developer**, I want to call `router.listen()` so that the router attaches to the Compute `fetch` event and handles all incoming HTTP requests automatically.
- **As an edge application developer**, I want automatic CORS preflight handling (`autoCorsPreflight` config, `OPTIONS *` handler) so that browsers receive correct `Access-Control-Allow-*` response headers without boilerplate.
- **As an edge application developer**, I want automatic cookie parsing (`parseCookie` config) so that incoming `Cookie` headers are available as structured data on the request object.
- **As an edge application developer**, I want route parameter extraction (`extractRequestParameters` config, `path-to-regexp`) so that URL segments like `/users/:id` are parsed and accessible on the request.
- **As an edge application developer**, I want typed extension of `EReq` and `ERes` via generics (`Router<MyReq, MyRes>`) so that custom request/response properties are type-safe across all handlers.
- **As an edge application developer**, I want built-in error handling (404 Not Found, 405 Method Not Allowed, 500 Internal Server Error) so that unmatched or failed requests return well-formed HTTP error responses without additional code.

## Business Rules

- A `Router` registers itself as the sole `fetch` event listener when `listen()` is called; all incoming requests are funnelled through a single handler chain. (inferred)
- Route handlers are evaluated **in registration order**; once a response has ended (`res.hasEnded`), further handlers in the stack are skipped.
- If no handler matches the request path, an `ErrorNotFound` (404) is thrown and the default error handler returns HTTP 404.
- If a path matches but the HTTP method does not, an `ErrorMethodNotAllowed` is thrown; with `auto405: true` (the default), the default error handler returns HTTP 405 with an `Allow` header listing the permitted methods.
- When `auto405` is `false` and a method mismatch occurs, the default error handler falls back to HTTP 404.
- Unhandled exceptions in the handler stack result in HTTP 500 with a JSON body `{ error: <message> }`.
- CORS preflight (`autoCorsPreflight`) returns HTTP 403 if the configured `trustedOrigins` list is empty. (inferred from code)
- CORS preflight returns HTTP 403 if the request's `origin` header is absent or does not case-insensitively match a trusted origin (unless `"*"` is configured). (inferred)
- When `trustedOrigins` includes `"*"`, the wildcard is forwarded as `Access-Control-Allow-Origin: *`; credentials-based requests will therefore fail per browser policy. (documented in code comment)
- `access-control-request-method` and `access-control-request-headers` request headers, when present during preflight, are echoed back as the corresponding `Access-Control-Allow-*` response headers.
- Path matching is delegated to `path-to-regexp`; matched path parameters are cached per pattern string (`pathMatcherCache`) to avoid redundant compilation. (inferred)
- The library targets Node.js ≥ 18 and the Fastly Compute JS runtime.
- `autoContentType` defaults to `false`; automatic `Content-Type` inference must be explicitly opted in. (inferred from default config)
- `parseCookie` defaults to `true`; cookie parsing is on by default unless disabled in `EConfig`.
