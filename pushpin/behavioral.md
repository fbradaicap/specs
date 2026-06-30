---
repo: pushpin
spec_type: behavioral
commit: e809e7c620218b758d1e71be3dfa3876cb096ac8
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 5399048df56eb38ec42bca7c4b7199dc36a40fb5854dcd7c4301849315faa20c
generated_at: 2026-06-30T14:42:45.985820059+02:00
generator: specsync
---

## API Contracts

Pushpin is a **reverse proxy** that exposes multiple synchronous interface types to clients and backend handlers. The protocols used are **HTTP/HTTPS**, **WebSocket**, and **ZeroMQ (ZMQ)**-based custom framing protocols (ZHTTP and M2 protocol from Mongrel2/GRIP).

### Client-facing HTTP/WebSocket proxy interface

Pushpin accepts inbound HTTP, HTTP long-polling, HTTP streaming, and WebSocket connections from downstream clients and forwards them to backend applications. The specific paths, methods, and routes are user-configured via the `routes` file (not statically defined in source). The three extracted endpoint identifiers correspond to **tnetstring/ZHTTP message field names** used in the ZMQ backend protocol, not HTTP paths:

| Identifier | Context |
|---|---|
| `headers` | Field in ZMQ request/response message envelope |
| `type` | Field in ZMQ message distinguishing packet kind (e.g., `credit`, `close`, `keep-alive`, `handoff-start`, `handoff-proceed`, `error`, `cancel`) |
| `more` | Field in ZMQ streaming response indicating more data follows |

### HTTP control/publish interface

The Docker image exposes the following ports:

| Port | Protocol | Purpose |
|---|---|---|
| 7999 | HTTP | Client-facing proxy port (forwards to backend app) |
| 5560 | ZMQ PUSH/PULL | Receive published messages from application |
| 5561 | HTTP | Receive published messages and commands (GRIP HTTP publish endpoint) |
| 5562 | ZMQ PUB/SUB | Receive published messages (subscribe channel) |
| 5563 | ZMQ REQ/REP | Receive commands |

### ZMQ backend handler protocol (ZHTTP / M2 protocol)

Backend handlers communicate with Pushpin over ZMQ using **tnetstring-serialized** message envelopes. The request/response message shapes evidenced in source tools are:

**Request message fields (Pushpin → handler):**

| Field | Type | Description |
|---|---|---|
| `from` | bytes | Return address of the originating pushpin instance |
| `id` | bytes/string | Connection/request identifier |
| `type` | string (optional) | Absent for data; `credit`, `keep-alive`, `handoff-proceed`, `error`, `cancel` for control |
| `uri` | bytes | Request URI |
| `headers` | list of `[name, value]` pairs | HTTP headers |
| `body` | bytes | Request body data |
| `more` | bool | Whether more body data follows (streaming input) |
| `credits` | int | Flow-control credit amount |

**Response message fields (handler → Pushpin):**

| Field | Type | Description |
|---|---|---|
| `from` | bytes | Handler instance identifier |
| `id` | bytes/string or list | Connection identifier(s) |
| `seq` | int | Sequence number for ordered delivery |
| `code` | int | HTTP status code (first response packet only) |
| `reason` | bytes | HTTP reason phrase |
| `headers` | list of `[name, value]` pairs | HTTP response headers |
| `body` | bytes | Response body chunk |
| `more` | bool | Whether more body data follows |
| `credits` | int | Flow-control credit granted to peer |
| `type` | string (optional) | Control packet type: `handoff-start`, `keep-alive`, `credit`, `close`, `error` |
| `content-type` | string (optional) | WebSocket frame content type |

---

## Event Schemas

Pushpin uses **ZeroMQ** as its internal/external messaging broker. No traditional message broker (Kafka, RabbitMQ) is present. The asynchronous messaging patterns evidenced are:

### ZMQ socket types and topology

| Socket | ZMQ Type | Direction | Purpose |
|---|---|---|---|
| `ipc://client-out` (or `/tmp/zhttp-test-out`) | PULL | Pushpin → handler | Delivers new HTTP/WS requests to handler |
| `ipc://client-out-stream` (or `/tmp/zhttp-test-out-stream`) | ROUTER/DEALER | Pushpin ↔ handler | Streaming/bidirectional mid-session messages |
| `ipc://client-in` (or `/tmp/zhttp-test-in`) | PUB/SUB | handler → Pushpin | Publishes responses back to proxy |
| `ipc://client` | REP/REQ | Pushpin ↔ handler | Request-reply mode (non-streaming) |

### Message type events (ZMQ payload "type" field values)

| Event type | Direction | Payload fields |
|---|---|---|
| _(absent — data packet)_ | both | `id`, `from`, `seq`, `uri`, `headers`, `body`, `more`, `credits` |
| `credit` | both | `id`, `from`, `seq`, `credits` |
| `keep-alive` | both | `id`, `from`, `seq` |
| `handoff-start` | handler → proxy | `id`, `from`, `seq` |
| `handoff-proceed` | proxy → handler | `id`, `from`, `seq` |
| `close` (WebSocket) | both | `id`, `from`, `seq`, `code` (optional) |
| `cancel` | proxy → handler | `id`, `from`, `seq` |
| `error` | proxy → handler | `id`, `from`, `seq`, `condition` |

---

## Input / Output Formats

### Serialization

- **ZMQ backend protocol**: [tnetstring](https://tnetstring.org/) encoding. Messages are prefixed with the routing address (for PUB sockets: `<address> T<tnetstring-payload>`; for REP sockets: `T<tnetstring-payload>`).
- **Client-facing HTTP interface**: Standard HTTP/1.x with arbitrary content negotiated between client and backend application. Pushpin is transparent to payload format.
- **GRIP HTTP publish endpoint (port 5561)**: HTTP with **JSON** body (GRIP protocol).

### Content types evidenced in handler examples

| Content-Type | Context |
|---|---|
| `text/plain` | Default handler response body |
| Configurable via `Content-Type` header passthrough | WebSocket and streaming responses |

### Streaming / flow control

- Streaming responses use a **credit-based flow control** mechanism: the handler receives `credits` (byte count) from Pushpin; it may only send that many bytes of body before receiving more credits.
- The `more: True` field on a response packet signals that additional body packets follow; absence (or `more: False`) signals end of response.
- Sequence numbers (`seq`) ensure in-order delivery for streaming sessions.

### Pagination

_Not determinable from code._

---

## Error Handling

### ZMQ protocol-level errors

Error conditions are communicated back to handlers as a ZMQ message with `type: "error"` and a `condition` field indicating the error reason (evidenced in `handoffhandler.py`):

```
{ "type": "error", "condition": "<string>", "id": ..., "from": ..., "seq": ... }
```

### HTTP status codes evidenced in handler tooling

| Code | Meaning |
|---|---|
| 200 | Successful response |
| 101 | WebSocket upgrade / Switching Protocols |

Other HTTP status codes may be proxied transparently from backend to client; no additional status codes are evidenced in the source snapshot.

### Validation behaviour

- Handlers are expected to validate `type` field presence to distinguish data vs. control packets; packets with unexpected types are silently dropped in the reference tool implementations.
- Session de-duplication: if an `id` already exists in active sessions, subsequent initial-request packets for the same `id` are ignored (logged as "session already active").
- Sequence numbers are validated implicitly by ordered delivery; out-of-order behaviour is _not determinable from code._

---

## Versioning

No URI versioning, header-based versioning, or schema evolution strategy is evidenced in the source code or configuration. The package version is `1.42.0-dev` (from `Cargo.toml`). The Docker image and compose file reference release `1.41.0`.

The GRIP/ZMQ protocol does not carry an explicit version field in the evidenced message schemas. Any versioning of the tnetstring wire format is _not determinable from code._
