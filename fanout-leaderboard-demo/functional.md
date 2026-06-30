---
repo: fanout-leaderboard-demo
spec_type: functional
commit: a12834febfc6421401e713b919d3a7edaabf81be
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 654db5a3047c6cc5c8633c79339ca38316e560d3fd644fcd765daa17e6033702
generated_at: 2026-06-30T14:56:31.067691311+02:00
generator: specsync
---

## Business Purpose

This service provides a real-time leaderboard capability, allowing multiple clients to view and update player scores simultaneously with live push updates. It exists to demonstrate Fastly Fanout as a GRIP proxy for broadcasting score changes to all connected devices without polling. The backend maintains leaderboard state in SQLite and publishes updates through Fanout's Server-Sent Events (SSE) mechanism.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Real-time gaming leaderboard.
- **Core entities / aggregates:**
  - `Board` — a named leaderboard identified by `boardId`.
  - `Player` — a participant on a board, identified by `playerId`, owning a `name` and `score`.
- **Neighbouring contexts:**
  - **Upstream:** Fastly Fanout / GRIP proxy (edge layer in `edge/`) routes and holds streaming connections, forwarding API calls to this origin.
  - **Downstream / infrastructure:** SQLite (persistence), Fastly Fanout publish channel (`board-<id>`) for broadcasting score updates to subscribers.

## Use Cases / User Stories

- **As a viewer**, I want to retrieve the current top players on a leaderboard so that I can see the rankings. → `GET /:boardId/` (JSON response when SSE is not requested; returns top 5 players.)
- **As a viewer**, I want to subscribe to a live leaderboard stream so that my display updates automatically when scores change. → `GET /:boardId/` with `Accept: text/event-stream`; the service opens a GRIP hold-stream on channel `board-<boardId>`.
- **As a client**, I want to fetch a single player's current score so that I can display their individual status. → `GET /:boardId/players/:playerId/`
- **As a game client**, I want to increment a player's score so that the leaderboard reflects new activity. → `POST /:boardId/players/:playerId/score-add/`

## Business Rules

- A request for a non-existent `boardId` or `playerId` returns HTTP 404. (Enforced in all three handlers.)
- SSE streaming (`text/event-stream`) is only permitted when the request is proxied through a GRIP intermediary (`req.grip.isProxied`); otherwise the service returns HTTP 406. (Enforced.)
- When the GRIP proxy requires signed requests (`req.grip.needsSigned`), unsigned requests for SSE are rejected with HTTP 501. (Enforced.)
- The leaderboard view returns only the **top 5 players** for a given board. (Hardcoded in `Player.getTopForBoard(board, 5)`.)
- Score changes are atomic increments via `player.scoreAdd()`; the updated score is returned in the response. (Inferred — the exact increment value is not visible in the sampled code.)
- A player must belong to the specified board; a player found under a different board is treated as not found (404). (Inferred — `Player.get(playerId, boardId)` takes both keys.)
- Published SSE updates are scoped to the per-board channel `board-<boardId>`, so subscribers only receive updates for the board they are watching. (Enforced in the SSE handler.)
