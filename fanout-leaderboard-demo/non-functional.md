---
repo: fanout-leaderboard-demo
spec_type: non_functional
commit: a12834febfc6421401e713b919d3a7edaabf81be
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 654db5a3047c6cc5c8633c79339ca38316e560d3fd644fcd765daa17e6033702
generated_at: 2026-06-30T14:56:31.067691311+02:00
generator: specsync
---

## Performance

- The backend is a Node.js (≥22.9.0) Express 5.x application; no explicit request timeout configuration is evident in the source snapshot.
- SQLite is used as the persistence layer. SQLite enforces serialised write access, which limits write throughput under concurrent load. No connection-pool configuration is evident (SQLite's `sqlite`/`sqlite3` driver does not pool connections in the traditional sense).
- Leaderboard reads return the top 5 players (`Player.getTopForBoard(board, 5)`); result-set size is bounded by design.
- No HTTP-level response caching headers (e.g., `Cache-Control`) or in-process caching layer (e.g., Redis, in-memory LRU) are evident in the source.
- Real-time fan-out is offloaded entirely to Fastly Fanout/GRIP: the origin holds no long-lived SSE connections itself; it issues GRIP hold-stream instructions and returns immediately, reducing per-connection resource cost on the origin to near zero.
- The Webpack production build (`NODE_ENV=production webpack`) minifies and bundles the React client, reducing client-side asset payload.
- No explicit thread-pool tuning, `--max-old-space-size`, or `cluster`/worker-thread configuration is evident.

## Scalability

- The edge layer is a Fastly Compute (WebAssembly) application deployed on Fastly's globally distributed network; it scales horizontally by the platform.
- The origin backend is documented as running on Google App Engine (a horizontally scalable PaaS), but no `app.yaml` or App Engine configuration is present in the snapshot; replica/instance count and autoscaling parameters are _Not determinable from code._
- The origin process is single-threaded Node.js with no `cluster` module usage evident; horizontal scaling of the origin would require multiple instances behind a load balancer.
- SQLite is file-based and does not support multi-process concurrent writes; horizontal scaling of the origin beyond a single writer instance would require replacing SQLite with a networked database. This represents a hard scalability constraint.
- Fan-out message delivery scales via Fastly Fanout: the origin publishes once per score update and Fanout distributes to all connected clients, decoupling origin throughput from subscriber count.
- No partitioning or sharding strategy is evident.

## Security

- **GRIP proxy authentication**: The SSE endpoint (`GET /:boardId/`) explicitly checks `req.grip.needsSigned && !req.grip.isSigned` and returns HTTP 501 if the GRIP proxy request is not signed, preventing unauthenticated proxies from opening hold-stream channels.
- **GRIP proxy presence check**: The same endpoint returns HTTP 406 if the request is not proxied (`!req.grip.isProxied`), ensuring SSE is only served through the Fastly Fanout intermediary.
- **Transport security**: TLS is enforced at the Fastly edge for public traffic; the origin-to-edge link security depends on deployment configuration, which is _Not determinable from code._
- **AuthN/AuthZ for write endpoints**: No authentication or authorisation middleware is evident on `POST /:boardId/players/:playerId/score-add/`; any client that can reach the endpoint can increment scores. This is consistent with the demo nature of the application but represents an open write surface.
- **Secrets handling**: The server startup script uses `--env-file-if-exists=.env` to load environment variables from a `.env` file if present; no secrets-manager integration or secret-rotation mechanism is evident.
- **Input validation**: Route parameters (`boardId`, `playerId`) are validated implicitly by database lookup (returning 404 if not found); no explicit schema validation or sanitisation middleware (e.g., `express-validator`) is evident.
- **Security headers / CSRF protection**: No Helmet, CORS, or CSRF middleware is evident in the source snapshot.
- **Dependency supply-chain**: All packages are pinned to semver ranges (e.g., `^`); no lockfile audit configuration is evident in the snapshot.

## Observability

- **Logging**: No structured logging library (e.g., Winston, Pino) or log-level configuration is evident; Express 5 default request logging and `console.*` output (if any) would be the only log streams.
- **Metrics**: No metrics instrumentation (e.g., Prometheus `prom-client`, StatsD) is evident.
- **Tracing**: No distributed tracing integration (e.g., OpenTelemetry, Jaeger) is evident.
- **Health/readiness endpoints**: No dedicated `/health`, `/ready`, or `/live` endpoint is evident in the source snapshot.
- Edge-level observability (Fastly real-time log streaming, observability dashboards) may be configured in the Fastly service but is _Not determinable from code._

## Reliability

- **Retries**: No client-side or server-side retry logic is evident in the source snapshot.
- **Circuit breakers**: No circuit-breaker pattern (e.g., `opossum`) is evident.
- **Timeouts**: No explicit HTTP server timeout (`server.timeout`, `server.keepAliveTimeout`) or database query timeout configuration is evident.
- **Idempotency**: The score-add endpoint (`POST /:boardId/players/:playerId/score-add/`) is not idempotent by design (each call increments the score); no idempotency key mechanism is evident.
- **SSE reconnection resilience**: GRIP/Fanout handles reconnecting subscribers transparently at the edge; the origin does not need to maintain per-client state across reconnects.
- **Database durability**: SQLite provides ACID guarantees for single-process access. No WAL-mode configuration or backup strategy is evident.
- **Process supervision**: `nodemon` is used in development only; production process supervision (e.g., systemd, App Engine instance restart policy) depends on the deployment platform and is _Not determinable from code._
- **Graceful shutdown**: No `SIGTERM`/`SIGINT` handler or in-flight request draining is evident in the source snapshot.
