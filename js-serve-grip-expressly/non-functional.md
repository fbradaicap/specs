---
repo: js-serve-grip-expressly
spec_type: non_functional
commit: 60e0350603e451b5ab32cc8883661b7b3e574c8c
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9232c1c7722e84c090d6d59cc77563f4961711f698c88abd545354e81f5d83f4
generated_at: 2026-06-30T14:52:34.254826147+02:00
generator: specsync
---

## Performance

_Not determinable from code._

No explicit latency budgets, throughput targets, connection pool sizes, thread pool configurations, or caching layers are defined in the source code or configuration files. The library delegates request/response handling entirely to the `@fastly/expressly` runtime and the upstream Fastly Compute platform; any performance characteristics are inherited from that environment rather than configured within this package.

## Scalability

_Not determinable from code._

This package is a library/middleware adapter, not a deployed service with replica counts, autoscaling policies, or partitioning configuration. Horizontal scaling is implicitly inherited from the Fastly Compute@Edge platform on which consuming services run. The implementation is stateless at the library level — per-request state is held in `WeakMap` caches (`ExpresslyApiRequest._map`, `ExpresslyApiResponse._map`) that are scoped to the request object lifetime and do not persist across invocations.

## Security

**GRIP signature verification**
The library delegates GRIP proxy signature validation to `@fanoutio/serve-grip` and `@fastly/grip-compute-js`. Requests arriving via a Fastly Fanout proxy carry a signed header; `ServeGrip` middleware verifies this signature before populating `req.grip.isProxied`. Unauthenticated or incorrectly signed requests are not treated as proxied.

**Secret / credential handling**
The GRIP URL (including `iss` claim and API token supplied as `key=<api_token>`) is passed at construction time as a configuration string or object. No secret-store integration or environment-variable injection pattern is prescribed by the library itself; secret handling is the responsibility of the consuming application. The README illustrates embedding credentials inline in the GRIP URL, which is a (target) anti-pattern for production use.

**Transport security**
The documented GRIP URL scheme uses `https://fanout.fastly.com/…`, enforcing TLS in transit to the Fastly Fanout publisher endpoint. No explicit certificate-pinning configuration is present.

**Input validation**
WebSocket-over-HTTP requests are validated by the `serve-grip` middleware: the `wsContext` is only populated when the incoming request passes GRIP protocol checks. The example code explicitly checks `wsContext == null` and returns HTTP 400 for non-WebSocket requests, providing minimal input-level guarding.

**AuthN/AuthZ**
No application-level authentication or authorisation middleware is included in this library. Consumer applications are responsible for adding their own AuthN/AuthZ layers on top of the `ServeGrip` middleware.

## Observability

**Debug logging**
The `debug` package (v4) is a runtime dependency. Diagnostic output can be enabled at the consumer level by setting the `DEBUG` environment variable to the appropriate namespace. No structured logging framework is included.

**Metrics**
_Not determinable from code._

**Distributed tracing**
_Not determinable from code._

**Health / readiness endpoints**
_Not determinable from code._ This is a library; no dedicated health or readiness HTTP handlers are defined.

## Reliability

**Error handling**
The `ServeGrip` middleware propagates publisher errors as thrown exceptions. The README example demonstrates wrapping `publisher.publishHttpStream` / `publisher.publishFormats` calls in `try/catch` blocks and returning HTTP 500 with error detail, providing a basic failure path.

**Retries and circuit breakers**
_Not determinable from code._ No retry logic, circuit-breaker, or backoff configuration is present in this library; these concerns are delegated to `@fanoutio/serve-grip` and the Fastly Compute platform networking layer.

**Idempotency**
_Not determinable from code._ Publish operations are fire-and-forget within the handler; no idempotency keys or deduplication mechanisms are implemented.

**Stateless failure isolation**
Because per-request objects are stored only in `WeakMap` instances tied to request object lifetimes, a failed or dropped request does not corrupt state for concurrent or subsequent requests. This provides implicit isolation at the request level.

**Availability**
Availability characteristics are entirely determined by the Fastly Compute@Edge platform and the upstream Fanout/Pushpin infrastructure; nothing in this library introduces additional single points of failure.
