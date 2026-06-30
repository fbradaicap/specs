---
repo: graphql-compute-example
spec_type: functional
commit: acd3472b4ad7c13c9e26622246735fa9ff16eb3f
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 840b244c7e882e1a2625320741063dbcea7474fb6eb0c568a8f321b43f1df774
generated_at: 2026-06-30T14:49:39.187192395+02:00
generator: specsync
---

## Business Purpose

This service is a reference implementation demonstrating how to run a GraphQL API server on Fastly's Compute edge platform. It processes GraphQL queries (e.g., fetching users and posts) directly at the network edge, moving query execution as close to end-users as possible. Its primary business value is improving API response latency and reducing origin load by enabling edge-side GraphQL query processing and caching.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge GraphQL API Gateway / example application.
- **Core domain entities:** `User` (with fields: `id`, `name`, `email`, `address.city`, `company.name`) and `Post` (with fields: `id`, `title`, `userId`, `body`), as evidenced by the README example queries.
- **Neighbouring contexts:** Acts as an edge intermediary in front of an upstream origin data source (implied by the README's mention of caching requests sent to origin). The actual upstream origin service is `_Not determinable from code._`

## Use Cases / User Stories

- **As an API consumer**, I want to query a list of users (including their id, name, email, city, and company name) so that I receive only the fields I need in a single request.
- **As an API consumer**, I want to query a list of posts (including id, title, userId, and body) so that I can retrieve post data efficiently without over-fetching.
- **As a platform engineer**, I want GraphQL query execution to run at the Fastly edge so that latency is minimised and origin load is reduced through edge caching.
- **As a developer**, I want a deployable example of GraphQL on Fastly Compute (`fastly compute publish`) so that I can use it as a starting point for building production edge GraphQL services.

> Note: No explicit HTTP endpoint paths are detected in extracted facts; routing is handled at runtime via `@fastly/expressly`. Specific route paths are `_Not determinable from code._`

## Business Rules

- Requests must be valid GraphQL queries conforming to the schema that exposes `users` and `posts` query roots (inferred from README examples and `graphql-helix` usage).
- The service must respond only to the fields explicitly requested in a GraphQL query — no over-fetching of fields beyond what the client specifies (intrinsic GraphQL rule; inferred).
- The build pipeline requires Node.js ≥ 18.0.0 as enforced by the `engines` field in `package.json`.
- The service is compiled to a WebAssembly binary (`bin/main.wasm`) via `@fastly/js-compute` before deployment; the runtime artefact is not raw JavaScript (inferred from build scripts).
- Deployment is exclusively to the Fastly Compute platform via `fastly compute publish`; no alternative deployment targets are defined.
- Origin caching behaviour (TTL, cache keys, surrogate headers) is `_Not determinable from code._`
- Authentication or authorisation requirements on incoming GraphQL requests are `_Not determinable from code._`
