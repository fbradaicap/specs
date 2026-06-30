---
repo: expressly
spec_type: non_functional
commit: d15cb937392edb46d2e39d406240dc6fdbf78c6b
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4e4d8e267b36915bdf0c4034e74b77d1ec77facdcabde45d5d705636dbf3fd75
generated_at: 2026-06-30T14:53:20.999053512+02:00
generator: specsync
---

## Performance

- **Runtime target:** Fastly Compute edge (via `@fastly/js-compute ^3.7.0`). Request processing is inherently single-threaded and event-driven on the Fastly Compute platform; there is no traditional thread-pool configuration exposed in this codebase.
- **Path matching cache:** A module-level `Map<string, Function>` (`pathMatcherCache`) is used to memoize compiled `path-to-regexp` matchers, avoiding repeated compilation cost on hot paths.
- **Cookie parsing:** Controlled by the `parseCookie` config flag (default `true`); can be disabled to reduce per-request overhead.
- **Timeouts:** _Not determinable from code._
- **Connection pools / concurrency limits:** _Not determinable from code._ (Managed by the Fastly Compute runtime, not this library.)
- **Caching:** No application-level response caching is configured within the library itself. Edge caching behaviour is delegated to the Fastly platform.

## Scalability

- **Execution model:** Deployed as a Fastly Compute edge service. Each invocation is stateless by design — no in-process shared state beyond the module-level `pathMatcherCache` (which is effectively per-isolate/instance). Horizontal scaling is handled transparently by the Fastly CDN infrastructure.
- **Statelessness:** The router holds only configuration and handler arrays set at startup; per-request state lives in `ERequest`/`EResponse` objects scoped to the `handler()` call. This supports safe horizontal scaling.
- **Replica counts / autoscaling:** _Not determinable from code._ Governed by Fastly platform capacity, not by this library.
- **Partitioning / sharding:** _Not determinable from code._
- **Node.js engine constraint:** `>=18` (declared in both `package.json` manifests); relevant when running tests or the documentation site locally.

## Security

- **CORS / preflight handling:** Built-in `autoCorsPreflight` option. When enabled, an `OPTIONS *` handler is registered that:
  - Returns `403` if no trusted origins are configured or if the `origin` request header does not match an entry in `trustedOrigins`.
  - Reflects `Access-Control-Request-Method` and `Access-Control-Request-Headers` back as the corresponding `Allow` response headers only for validated origins.
  - Supports explicit allowlist or wildcard (`*`) origin policy (with the documented caveat that `*` is incompatible with credentialled requests).
- **AuthN/AuthZ:** _Not determinable from code._ No authentication or authorisation middleware is included in the library; these are expected to be supplied by the consuming application via `.use()`.
- **Transport security (TLS):** Inherited from the Fastly Compute platform; not configurable at the library level.
- **Cookie handling:** Uses the `cookie ^1.1.1` package for parsing. Cookie parsing is opt-out (`parseCookie: true` by default).
- **Input validation:** Path matching is performed via `path-to-regexp ^6.2.1`. No additional schema-level input validation is present in the library core.
- **Secrets handling:** _Not determinable from code._ No secrets management is present in the library.
- **HTTP method enforcement:** `auto405` (default `true`) causes the default error handler to return `405 Method Not Allowed` with an `Allow` header when a path is matched but the method is not registered, preventing unintended method exposure.
- **Error detail exposure:** The default error handler returns `{ error: err.message }` as JSON for unhandled errors (HTTP 500), which may leak internal error messages (target: consuming applications should override this handler in production).

## Observability

- **Logging:** `console.error(err)` is called inside the default error handler for unhandled (non-404/405) errors. No structured logging framework is configured.
- **Metrics:** _Not determinable from code._
- **Tracing:** _Not determinable from code._
- **Health / readiness endpoints:** _Not determinable from code._ No built-in health check route is registered by the library; consuming applications must add one.
- **Events (internal):** Uses `mitt ^3.0.1` for an internal event emitter (visible in the `ERes` type tests via `res.on('finish', ...)`), enabling consumers to hook into response lifecycle events, but no external observability sink is wired up.

## Reliability

- **Error isolation:** All request handler and middleware execution is wrapped in a `try/catch` in the top-level `handler()` method. Uncaught errors are routed through the error-handler stack, with a default handler appended as a safety net.
- **Graceful 404/405 handling:** `ErrorNotFound` and `ErrorMethodNotAllowed` are typed, caught, and mapped to appropriate HTTP status codes (404 / 405) with correct `Allow` header population, preventing unhandled rejections from surfacing as 500s.
- **Retries:** _Not determinable from code._
- **Circuit breakers:** _Not determinable from code._
- **Idempotency:** _Not determinable from code._
- **Response finalisation guard:** `res.hasEnded` is checked before each handler in the pipeline, preventing handlers from running after a response has already been committed and avoiding double-send errors.
- **Availability / SLA:** _Not determinable from code._ Availability is governed by the Fastly Compute platform SLA, not by this library.
- **State recovery:** Because the service is stateless and each request runs in an isolated invocation context on the Fastly platform, there is no persistent state to recover. Failure of one invocation does not affect others.
