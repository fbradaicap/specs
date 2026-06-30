---
repo: pushpin
spec_type: functional
commit: e809e7c620218b758d1e71be3dfa3876cb096ac8
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 5399048df56eb38ec42bca7c4b7199dc36a40fb5854dcd7c4301849315faa20c
generated_at: 2026-06-30T14:42:45.985820059+02:00
generator: specsync
---

## Business Purpose

Pushpin is a reverse proxy server (authored by Fastly) that enables realtime web services by holding long-lived client connections open while communicating with backend applications via short-lived HTTP requests using the GRIP (Generic Realtime Intermediary Protocol) protocol. It supports WebSocket, HTTP streaming, and HTTP long-polling without requiring clients or backends to use a proprietary wire protocol. It exists to decouple connection management from backend application logic, allowing backends to be written in any language.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Realtime connection proxy / connection management gateway.
- **Core domain entities / aggregates:**
  - **Connection** — a persistent client connection (HTTP long-poll, HTTP streaming, or WebSocket) held open by the proxy.
  - **Session** — a tracked request/response lifecycle between the proxy and a backend, including credit-based flow control state (evidenced in `streamhandler.py`, `streamposthandler.py`, `holdhandler.py`).
  - **Route** — a mapping rule from an inbound request to a backend target (evidenced by the `routes` configuration file asset in `Cargo.toml`).
  - **Message/Publication** — a unit of data pushed to held connections (evidenced by the `pushpin-publish` binary and the ZMQ publish/subscribe sockets).
- **Relationships to neighbouring contexts:**
  - **Upstream (clients):** Arbitrary HTTP/WebSocket clients connecting on the exposed HTTP port (default 7999 per Dockerfile).
  - **Downstream (backends):** Backend web applications speaking the GRIP protocol over short-lived HTTP or ZMQ (ZMQ PULL/PUB/REP sockets on ports 5560–5563; HTTP on port 5561).
  - **Mongrel2 integration (upstream adapter):** The `m2adapter` component sits behind a Mongrel2 instance and converts its protocol to the internal ZMQ-HTTP (`zhttp`) protocol, making Mongrel2 an optional upstream source.

## Use Cases / User Stories

- **As a backend application developer**, I want to instruct Pushpin (via a GRIP response header) to hold a client's HTTP connection open, so that I can push data to that client later without managing persistent connections myself.
- **As a backend application developer**, I want to publish messages to one or more held connections via ZMQ (ports 5560, 5562) or HTTP (port 5561), so that I can implement server-sent events or long-polling feeds.
- **As a backend application developer**, I want to establish and communicate over WebSocket connections proxied through Pushpin, so that my backend only needs to handle short HTTP interactions while Pushpin manages the WebSocket lifecycle (evidenced by `wsechohandler.py`, `streamhandler.py` WebSocket branch).
- **As a backend application developer**, I want to stream large HTTP responses incrementally using a credit-based flow-control mechanism, so that I can send files or continuous data to clients without buffering the full response (evidenced by `streamhandler.py`, `streamposthandler.py`).
- **As a backend application developer**, I want to use HTTP long-polling where Pushpin holds the connection and releases it when a message is published, so that I can implement poll-based realtime APIs transparently (evidenced by `holdhandler.py`).
- **As an operator**, I want to configure routing rules (via the `routes` file) to direct inbound requests to different backend targets, so that a single Pushpin instance can front multiple backend services.
- **As an operator**, I want to connect one or more Mongrel2 instances to the same `m2adapter`, so that I can scale the frontend HTTP tier independently of the proxy logic (evidenced by README of `m2adapter`).
- **As a developer or operator**, I want to publish messages directly from the command line using `pushpin-publish`, so that I can test or trigger realtime updates without writing application code.

## Business Rules

- A Mongrel2 instance **must not** be linked to more than one `m2adapter` instance; one `m2adapter` may serve multiple Mongrel2 instances (stated in `m2adapter` README).
- Backend communication must use the GRIP protocol over short-lived HTTP requests or ZMQ sockets; Pushpin does not expose a proprietary protocol to end clients.
- Client-facing HTTP/WebSocket content is fully controlled by the backend — Pushpin is transparent to the wire protocol between client and server.
- Flow control for streaming responses is credit-based: the backend may only send as many bytes as credits granted by the proxy; additional credits are issued incrementally (evidenced in `streamhandler.py` and `streamposthandler.py`).
- ZMQ session messages carry a monotonically increasing sequence number (`seq`); out-of-order or missing sequence numbers indicate a protocol error (inferred from sequence tracking in all handler tools).
- A connection/session ID must be unique per active session; duplicate IDs on new incoming requests are rejected with "session already active" (evidenced in `streamhandler.py`, `keephandler.py`).
- The first packet in a new session must be a data packet (no `type` field); typed packets (e.g. `keep-alive`, `credit`, `close`) on an unknown session are discarded (evidenced in `wsechohandler.py`, `keephandler.py`).
- WebSocket connections are accepted by responding with HTTP 101 and a `credits` grant before data frames are exchanged (evidenced in `streamhandler.py` WebSocket branch).
- Connections have a TTL and are expired if not refreshed within the configured interval (evidenced by `CONN_TTL` / `EXPIRE_INTERVAL` constants in `holdhandler.py`). (inferred as a configurable parameter in production)
- TLS is supported via both rustls and OpenSSL (both present as dependencies in `Cargo.toml`); native CA certificates are loaded for outbound connections (inferred from `rustls-native-certs` dependency).
- JWT-based authentication is supported, likely for GRIP publish authorization (inferred from `jsonwebtoken` dependency in `Cargo.toml`).
- The service must run as a non-root user (UID 1001) in containerized deployments (evidenced in `docker/Dockerfile`).
