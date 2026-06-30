---
repo: sse-demo
spec_type: functional
commit: f6ccb158355eabeb83eb90bdd788b11e1d9e6fed
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 761467fdcd341a4f294991cdbe223e666b92f385ef65f30b48c5918a47ca804a
generated_at: 2026-06-30T14:52:55.066424752+02:00
generator: specsync
---

## Business Purpose

This service is a demonstration application that showcases **Server-Sent Events (SSE)** delivered through a Fastly CDN edge layer. It exists to illustrate how real-time, server-pushed data streams can be efficiently distributed at the edge, using a simple Express.js backend that generates and streams data to connected browser clients. The `gaussian` dependency suggests the streamed data is synthesised using a Gaussian (normal) distribution, making it a realistic-looking simulated data feed for demonstration purposes.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Real-time data streaming / edge-delivery demonstration.
- **Core entities/aggregates:**
  - Simulated data stream (Gaussian-distributed numeric values generated server-side).
  - SSE connection lifecycle (open, streaming, close).
- **Neighbouring contexts:** No upstream or downstream service integrations are evident. The primary external relationship is with the **Fastly CDN layer**, which sits between this origin server and browser clients to cache/distribute the SSE stream at the edge.

## Use Cases / User Stories

- **As a developer**, I want to open a browser page served by this demo so that I can observe a live, server-pushed data stream without polling.
- **As a developer**, I want the SSE stream to carry Gaussian-distributed data values so that the demo resembles a realistic real-time signal (e.g. sensor readings or metrics).
- **As a Fastly solutions engineer**, I want a running origin server that emits SSE so that I can demonstrate Fastly's edge-streaming and long-lived connection capabilities to prospects.
- **As a developer**, I want a static front-end (files under `public/`) served alongside the streaming endpoint so that no separate hosting is needed for the demo UI.
- **As an operator**, I want environment-specific configuration loaded via `dotenv` so that port numbers or API keys can be adjusted without code changes.

## Business Rules

- **SSE protocol compliance:** The server must emit responses with `Content-Type: text/event-stream` and keep the HTTP connection open for the lifetime of the client session. _(inferred from SSE convention and service purpose.)_
- **Data generation:** Streamed values are drawn from a Gaussian distribution (mean and variance configurable or hardcoded via the `gaussian` library). _(inferred from dependency.)_
- **Environment configuration:** Runtime parameters (e.g. listening port, any Fastly-specific tokens) must be supplied through a `.env` file or environment variables; defaults may be absent and the service could fail to start without them. _(inferred from `dotenv` usage.)_
- **No persistence:** There are no database models or event-bus integrations, so no data is persisted; the stream is purely ephemeral and generated on-the-fly.
- **Single-process origin:** The server is started with a single `node server/server.js` command; horizontal scaling behaviour is `_Not determinable from code._`
- **Deployment target:** The `deploy` script targets Google Cloud App Engine (`gcloud app deploy`), indicating the origin is expected to run there, with Fastly in front. _(inferred from `package.json` deploy script.)_
