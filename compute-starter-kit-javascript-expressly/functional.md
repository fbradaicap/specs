---
repo: compute-starter-kit-javascript-expressly
spec_type: functional
commit: d0945ec18eb2d221a3d3da811ba45d7f46dfd215
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 07c150c73a3ef746e559192e517f0af4480943c215c8af3ad7b45210ea33efec
generated_at: 2026-06-30T14:51:30.820608466+02:00
generator: specsync
---

## Business Purpose

This service is a starter-kit template for building edge-side JavaScript applications on Fastly Compute using the Expressly routing framework. It demonstrates how to handle HTTP routing and middleware at the edge, generating synthetic responses without requiring any upstream backends. It exists to accelerate developer onboarding to the Fastly Compute platform by providing a working, deployable baseline application.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge request routing / Fastly Compute bootstrapping.
- **Core entities/aggregates:** HTTP request/response cycle handled entirely at the edge; no persistent domain entities or aggregates are modelled.
- **Neighbouring contexts:** None evident — the service is self-contained and generates synthetic responses; it has no upstream backend dependencies or downstream consumers defined in this snapshot.

## Use Cases / User Stories

- As a **developer new to Fastly Compute**, I want a pre-built JavaScript project scaffold so that I can quickly understand routing and middleware patterns at the edge without writing boilerplate from scratch.
- As a **developer**, I want to run the application locally (`npm run start`) so that I can iterate and test edge logic in a development environment before deployment.
- As a **developer**, I want to deploy the application to Fastly (`npm run deploy`) so that it runs as a live Fastly Compute service generating synthetic HTTP responses at the edge.
- As a **Fastly Compute service**, I want to route incoming HTTP requests using Expressly middleware so that different request paths are handled by the appropriate logic without proxying to an origin backend.

## Business Rules

- The application **must not require any configured backends**; all responses are generated synthetically at the edge. (Stated explicitly in README.)
- The compiled output **must be a WebAssembly binary** (`./bin/main.wasm`), produced by the `js-compute-runtime` build step — this is a hard platform requirement for Fastly Compute. (inferred from `package.json` build script.)
- The package is marked `"private": true`, meaning it is **not intended for publication to a public npm registry** and is only used as a local project template. (inferred from `package.json`.)
- Routing and middleware behaviour is governed by `@fastly/expressly ^2.0.0`; breaking changes from earlier Expressly versions are explicitly excluded by the semver range. (inferred from `package.json` dependency constraint.)
- The entry point for all application logic is `./src/index.js`; any routing or middleware must be registered there to be included in the compiled binary. (inferred from build script.)
