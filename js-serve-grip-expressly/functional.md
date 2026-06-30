---
repo: js-serve-grip-expressly
spec_type: functional
commit: 60e0350603e451b5ab32cc8883661b7b3e574c8c
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9232c1c7722e84c090d6d59cc77563f4961711f698c88abd545354e81f5d83f4
generated_at: 2026-06-30T14:52:34.254826147+02:00
generator: specsync
---

## Business Purpose

`@fastly/serve-grip-expressly` is a TypeScript library (npm package) that integrates the [js-serve-grip](https://github.com/fanout/js-serve-grip) GRIP middleware with the [Expressly](https://github.com/fastly/expressly) router framework for Fastly Compute. It enables Fastly Compute edge services to participate in the GRIP (Generic Realtime Intermediary Protocol) pub/sub mechanism, supporting both HTTP streaming and WebSocket-over-HTTP patterns through Fastly Fanout or a Pushpin proxy.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge real-time messaging integration — bridging Fastly Compute request handling with GRIP-based publish/subscribe infrastructure.
- **Core entities / abstractions this package owns:**
  - `ServeGrip` — middleware class that attaches GRIP context to requests and responses within an Expressly router.
  - `ExpresslyApiRequest` — adapter wrapping Expressly's `ERequest` to satisfy the `IApiRequest` interface expected by `@fanoutio/grip`.
  - `ExpresslyApiResponse` — adapter wrapping Expressly's `EResponse` to satisfy the `IApiResponse` interface expected by `@fanoutio/grip`.
- **Upstream dependencies (context providers):**
  - `@fanoutio/serve-grip` — core GRIP middleware logic (upstream).
  - `@fanoutio/grip` — GRIP protocol primitives and publisher (upstream).
  - `@fastly/expressly` — Fastly Compute edge router (upstream).
  - `@fastly/grip-compute-js` — Fastly Compute-specific GRIP transport (upstream).
- **Downstream consumers:** Application code running on Fastly Compute that uses this package as a middleware library; no downstream services are owned by this package itself.

## Use Cases / User Stories

- **As a Fastly Compute developer**, I want to add GRIP middleware to my Expressly router so that incoming requests from a Fastly Fanout or Pushpin proxy are automatically validated and enriched with GRIP context (`req.grip`, `res.grip`).
- **As a Fastly Compute developer**, I want to open an HTTP streaming connection to a named GRIP channel so that connected clients receive a long-lived stream of server-pushed messages. _(Illustrated by the `GET /api/stream` pattern in the README, using `gripInstruct.addChannel` and `gripInstruct.setHoldStream`.)_
- **As a Fastly Compute developer**, I want to publish a message to a GRIP channel so that all subscribed HTTP stream clients receive it in real time. _(Illustrated by the `GET /api/publish` pattern using `publisher.publishHttpStream`.)_
- **As a Fastly Compute developer**, I want to accept a WebSocket-over-HTTP connection and subscribe it to a GRIP channel so that bidirectional WebSocket communication is relayed through the GRIP proxy. _(Illustrated by the `POST /api/websocket` pattern using `wsContext.accept()` and `wsContext.subscribe`.)_
- **As a Fastly Compute developer**, I want to broadcast a message to all WebSocket-over-HTTP clients on a channel so that connected WebSocket clients receive the message. _(Illustrated by the `POST /api/broadcast` pattern using `publisher.publishFormats` with `WebSocketMessageFormat`.)_
- **As a Fastly Compute developer**, I want to run the integration locally against a Pushpin server so that I can develop and test GRIP-based functionality without deploying to Fastly.

## Business Rules

- The `ServeGrip` middleware **must** be registered via `router.use(serveGrip)` before any route handler that accesses `req.grip` or `res.grip`; this is the required initialisation order shown in all examples. (inferred)
- A Fastly backend named `grip-publisher` **must** be configured in `fastly.toml` and must be reachable at `fanout.fastly.com` (production) or `http://localhost:5561/` (local); the backend name is referenced explicitly in the GRIP URL and constructor options.
- HTTP streaming responses **must** check `req.grip.isProxied` before calling `startInstruct()`; if the request is not proxied through a GRIP proxy, the service should respond with a plain non-streaming response. (inferred from README example)
- WebSocket-over-HTTP handlers **must** verify that `req.grip.wsContext` is non-null and return HTTP 400 if it is absent, ensuring only legitimate GRIP-proxied WebSocket requests are processed.
- On a new WebSocket-over-HTTP connection (`wsContext.isOpening() === true`), the handler **must** call `wsContext.accept()` before subscribing to a channel; failure to accept would leave the connection unacknowledged. (inferred)
- `ExpresslyApiRequest` instances are cached per `ERequest` object via a `WeakMap` to ensure at most one adapter is created per request object. (inferred)
- `ExpresslyApiResponse` instances are similarly cached per `EResponse` object via a `WeakMap`. (inferred)
- Publish failures from `publisher.publishHttpStream` or `publisher.publishFormats` are surfaced as HTTP 500 responses with the error message and context in the body; this is the prescribed error-handling pattern for publish operations. (inferred from README example)
- The package targets the Fastly Compute runtime (`@fastly/expressly`, `@fastly/grip-compute-js`) and is not intended for use in Node.js or browser environments. (inferred)
