---
repo: fastly-js
spec_type: non_functional
commit: 2ff10a7eb745a90f2f38e13fe4aaa35ca962115e
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 6ba2660cb9f99488ebf2c1aac9e17f0bd432d3a66d76e4928f652363aa6e4357
generated_at: 2026-06-30T14:53:35.703318254+02:00
generator: specsync
---

## Performance

This repository is a **client-side JavaScript SDK** (npm package `fastly` v15.1.0) that wraps the Fastly REST API (`https://api.fastly.com`). It does not run as a server process and has no independently configurable performance parameters such as thread pools, connection pools, or server-side caches. All HTTP calls are delegated to the `superagent` ^6.1.0 library.

- **Timeouts:** No request timeout values are configured in the source samples or `package.json`. `superagent` defaults apply. _Not determinable from code._
- **Connection/thread pools:** Not applicable for a Node.js/browser client library; no pool configuration is present.
- **Caching:** No client-side response caching layer is implemented. The library does expose Fastly cache-settings and cache-condition APIs as thin wrappers, but these control remote Fastly service configuration, not local caching.
- **Bulk operations:** The `bulkUpdateAclEntries` method documents a maximum batch size of 1,000 entries per request, reflecting an upstream API constraint.
- **Pagination:** Several list endpoints (e.g., `getServiceLevelUsage`) support cursor-based pagination with a configurable `limit` parameter (default `'1000'`, maximum `10000`), reducing per-call payload size.
- **Build:** The distribution is pre-compiled from `src/` to `dist/` via Babel at pack time (`prepack` hook); no runtime transpilation overhead for consumers.

---

## Scalability

This is a **stateless client library**, not a deployable service. Scaling considerations apply to consumers of the package, not to the package itself.

- **Statelessness:** Each `ApiClient` instance holds only authentication state (the API token) and a base-path reference. Multiple instances can coexist independently in the same process.
- **Horizontal/vertical scaling:** Not applicable; the library is embedded in consumer applications.
- **Replica counts / autoscaling:** _Not determinable from code._
- **Partitioning/sharding:** Not applicable.
- **Multi-environment support:** The `_base_path_index` option present on every API call allows callers to select from the list of configured server base paths at runtime, enabling targeting of different Fastly API environments without re-instantiation.

---

## Security

- **Authentication:** All API classes use a `'token'` auth scheme. When running in a Node.js environment (`typeof window === 'undefined'`), each API class constructor automatically reads the `FASTLY_API_TOKEN` environment variable and calls `this.apiClient.authenticate(...)`. In browser environments, the token must be supplied explicitly by the caller.
- **Transport security:** All requests target `https://api.fastly.com` exclusively; HTTPS is enforced by the hardcoded base path across all API modules.
- **Secrets handling:** The API token is sourced from the `FASTLY_API_TOKEN` environment variable in server-side contexts, avoiding hard-coding credentials in source. No secrets-management framework (e.g., Vault, AWS SSM) is present; consumers are responsible for populating the environment variable securely.
- **Input validation:** Required parameters are validated with explicit guard checks (throwing `Error` if absent) before the HTTP call is dispatched. No schema-level or sanitization-level validation beyond presence checks is evident.
- **Browser file-system access:** `package.json` sets `"browser": { "fs": false }`, disabling Node.js `fs` module in browser bundles to prevent unintended file-system access.
- **Dependency security:** Runtime dependencies are minimal — only `superagent` ^6.1.0. All other dependencies are build-time (`devDependencies`). No security middleware (helmet, CORS, rate-limiting) is present, as this is a library, not a server.
- **AuthZ:** Authorization is delegated entirely to the Fastly API; the library performs no local authorization checks.

---

## Observability

- **Logging:** No logging framework or log statements are present in the source samples. _Not determinable from code._
- **Metrics:** No metrics instrumentation (e.g., Prometheus, StatsD, OpenTelemetry) is present. _Not determinable from code._
- **Tracing:** No distributed tracing instrumentation is present. _Not determinable from code._
- **Health/readiness endpoints:** Not applicable; the library exposes no HTTP server or health endpoint.
- **Error surfacing:** Errors are propagated to callers as rejected Promises, preserving the full HTTP response object (status code, headers, body) for programmatic inspection by consumers.

---

## Reliability

- **Retries:** No automatic retry logic is implemented within the library. `superagent` plugin-based retries are not configured. Retry responsibility lies with the consumer.
- **Circuit breakers:** Not implemented. _Not determinable from code._
- **Timeouts:** No per-request or global timeout is configured. _Not determinable from code._
- **Idempotency:** The library faithfully maps to REST semantics (GET, POST, PUT, PATCH, DELETE) but does not add idempotency keys or enforce idempotency behaviour beyond what the Fastly API provides natively.
- **Failure handling:** All API calls return Promises; rejections carry the HTTP error response, allowing consumers to implement their own error-handling and back-off strategies. Required-parameter validation throws synchronously before any network call, preventing unnecessary outbound requests.
- **Stale-if-error:** The `CacheSettingsApi` exposes a `stale_ttl` field representing the Fastly-side "stale if error" TTL, but this is a remote configuration parameter, not a local resilience mechanism.
- **Availability/recovery:** As a client library with no persistent state or storage, there is no recovery or failover logic to configure. Upstream Fastly API availability governs end-to-end reliability.
