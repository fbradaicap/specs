---
repo: microservice1
spec_type: functional
commit: 67a1a3cf92762515763f4e7d8cea0cfc4eeb30c1
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d79e551fd821da0289be60837d42cd2e5e852eb45e459af0626dff12f227911c
generated_at: 2026-06-30T16:51:42.511630771+02:00
generator: specsync
---

## Business Purpose

This service is a minimal Quarkus-based REST microservice that exposes a greeting endpoint. It appears to serve as a proof-of-concept, starter template, or hackathon scaffold rather than a production business capability. Its sole observable function is to respond to HTTP GET requests on `/hello` with a plain-text or JSON response.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Greeting / Health Check (template scope only).
- **Core domain entities/aggregates:** None — no persistent domain entities or aggregates are present.
- **Relationships to neighbouring contexts:** _Not determinable from code._ No upstream or downstream service integrations are evident.

## Use Cases / User Stories

- **As a client**, I want to call `GET /hello` so that I receive a plain-text "Hello" greeting (content type `text/plain`).
- **As a client**, I want to call `GET /hello` with a JSON-accepting header so that I receive the integer value `2` serialised as a JSON string (`application/json`). _(Note: both GET handlers are mapped to the same path `/hello`; runtime content-negotiation selects the response based on the `Accept` header.)_

## Business Rules

- The `/hello` endpoint must return HTTP 200 on a successful GET request (enforced by integration test `HelloResourceTest.testHelloEndpoint`).
- The plain-text variant of `GET /hello` returns the static string `"Hello"` (inferred from source); the test asserts the body equals `"Hello from Quarkus REST"`, which differs from the source implementation — indicating the test may be stale or target a different build artifact. (inferred)
- The JSON variant of `GET /hello` always returns the static string `"2"` regardless of any input; no parameters or request body are accepted. (inferred)
- No authentication, authorisation, or input validation rules are applied to any endpoint.
- No persistent state is written or read; the service is fully stateless.
- The service listens on port `8080` (defined in all Dockerfiles via `EXPOSE 8080` and `-Dquarkus.http.host=0.0.0.0`).
