---
repo: fanout-leaderboard-demo
spec_type: behavioral
commit: a12834febfc6421401e713b919d3a7edaabf81be
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 654db5a3047c6cc5c8633c79339ca38316e560d3fd644fcd765daa17e6033702
generated_at: 2026-06-30T14:56:31.067691311+02:00
generator: specsync
---

## API Contracts

Protocol: **HTTP/REST** (Express.js backend, served through Fastly Fanout/GRIP proxy at the edge).

All paths are mounted under a `boardsApiRouter`; the base prefix is not determinable from the sampled code alone, but the extracted endpoints indicate routes are served directly under `/:boardId/`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` | `/:boardId/` | Retrieve top players for a leaderboard, or open a Server-Sent Events stream for real-time updates | Accept header `text/event-stream` to request SSE; otherwise no body required | **JSON** `{ players: [{ id, name, score }] }` (top 5 players) **or** SSE stream held open by GRIP proxy (Content-Type: `text/event-stream`) |
| `GET` | `/:boardId/players/:playerId/` | Retrieve a single player's data for a given board | No body | **JSON** `{ id, name, score }` |
| `POST` | `/:boardId/players/:playerId/score-add/` | Increment a player's score by one unit | No body required | **JSON** `{ id, name, score }` (updated values) |

**Edge layer:** A separate Fastly Compute (WebAssembly) application in `edge/` intercepts applicable requests and hands them off to Fastly Fanout before forwarding to the Node.js backend origin.

---

## Event Schemas

**Broker / transport:** Fastly Fanout acting as a [GRIP (Generic Realtime Intermediary Protocol)](https://pushpin.org/docs/protocols/grip/) proxy. The backend uses `@fanoutio/grip` and `@fanoutio/serve-grip` to publish updates.

**Channel naming convention (evidenced in source):**

| Direction | Channel / Topic | Trigger | Payload shape |
|-----------|----------------|---------|---------------|
| Published (backend → Fanout → clients) | `board-<boardId>` | Score update via `POST /:boardId/players/:playerId/score-add/` (inferred from channel registration on the board endpoint) | _Not determinable from code._ (publish call is not visible in the sampled source) |
| Consumed (client subscribes) | `board-<boardId>` | Client issues `GET /:boardId/` with `Accept: text/event-stream` | SSE event stream; specific event names and field structure _Not determinable from code._ |

The GRIP channel `board-${board.id}` is registered in `server/api.js`; the publisher side (what is sent over that channel after a score update) is not present in the sampled source files.

---

## Input / Output Formats

- **Serialization:** JSON for all non-streaming responses. Express's `res.send(object)` serialises plain JavaScript objects as `application/json`.
- **SSE streaming:** `Content-Type: text/event-stream` when the client sends `Accept: text/event-stream` and the request is proxied through GRIP. The response body is held open by Fanout; the backend ends its own response immediately (`res.end()`).
- **Request bodies:** No request body is consumed by any of the three endpoints in the sampled code. The `POST /:boardId/players/:playerId/score-add/` endpoint derives all necessary information from path parameters alone.
- **Response envelope:** No wrapper envelope; responses are flat JSON objects:
  - Board listing: `{ players: Array<{ id, name, score }> }` (top 5 entries).
  - Single player / score-add: `{ id, name, score }`.
- **Pagination:** Fixed limit of 5 top players (`Player.getTopForBoard(board, 5)`); no pagination parameters are evidenced.
- **Content negotiation:** `GET /:boardId/` uses `req.accepts('text/event-stream')` to branch between SSE and JSON responses.

---

## Error Handling

Error responses use plain-text bodies (Express `res.send(string)`).

| Scenario | Status Code | Response body |
|----------|------------|---------------|
| Board not found | `404` | `Not found.\n` |
| Player not found (GET or POST) | `404` | `Not found.\n` |
| SSE requested but request is not proxied through GRIP | `406` | `text/event-stream requires GRIP proxy.\n` |
| SSE requested through GRIP but proxy signature is required and missing/invalid | `501` | `text/event-stream requires authenticated GRIP proxy.\n` |
| Successful resource retrieval or mutation | `200` | JSON object (see API Contracts) |

- **Validation:** Path parameters (`boardId`, `playerId`) are validated implicitly by looking up the entity in SQLite; no schema-level input validation is evidenced in the sampled code.
- **Unhandled/internal errors:** _Not determinable from code._ (no explicit error middleware is visible in the sampled source).

---

## Versioning

No URI versioning prefix, request header versioning, or schema evolution strategy is evidenced in the source code or configuration files. The `package.json` records an application version of `0.1.1` (root) and `0.1.0` (edge), but these are not surfaced in the API paths or headers.

_Not determinable from code._
