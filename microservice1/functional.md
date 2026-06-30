---
repo: microservice1
spec_type: functional
commit: 954d85981a924722137ae7edea41a6ffbf5b2444
model: claude-sonnet-4-6
prompt_version: v1
input_hash: d7c1d6d7e1a8acbea7e48f4ff300b586a4fb468527117a2559d0a34869f35a63
generated_at: 2026-06-30T16:00:13.254039078+02:00
generator: specsync
---

## Business Purpose

This service is a minimal "Hello World" REST microservice built on the Quarkus framework. It exposes a single HTTP endpoint that returns a plain-text greeting, serving as a skeleton or starter template for Quarkus-based REST services within the hackathonCapAmadeus organisation. It provides no substantive business capability beyond demonstrating basic Quarkus REST wiring.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Greeting / Demo — a trivial, self-contained context with no meaningful domain model.
- **Core entities/aggregates:** None; the service holds no state and manages no domain objects.
- **Neighbouring contexts:** None evident. No upstream or downstream service integrations are present.

## Use Cases / User Stories

- As a **client or developer**, I want to call `GET /hello` so that I can verify the service is running and receive a plain-text greeting response (`"Hello"`).
  - Mapped to: `GET /hello` → `HelloResource.hello()` → `HTTP 200 text/plain`

## Business Rules

- The `GET /hello` endpoint must return HTTP status `200` with a plain-text body. (Asserted by `HelloResourceTest` and its integration counterpart `HelloResourceIT`.)
- The response content type is `text/plain` (enforced via `@Produces(MediaType.TEXT_PLAIN)`).
- The endpoint is read-only; no mutation operations (POST, PUT, DELETE) are defined or permitted on this resource.
- No authentication, authorisation, or input validation rules are applied. (inferred — no security annotations or filters are present in the source.)
- No persistent state is maintained; the service is stateless with no database or messaging dependencies.
