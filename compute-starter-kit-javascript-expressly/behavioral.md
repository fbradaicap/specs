---
repo: compute-starter-kit-javascript-expressly
spec_type: behavioral
commit: d0945ec18eb2d221a3d3da811ba45d7f46dfd215
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 07c150c73a3ef746e559192e517f0af4480943c215c8af3ad7b45210ea33efec
generated_at: 2026-06-30T14:51:30.820608466+02:00
generator: specsync
---

## API Contracts

This service uses **HTTP (REST-style routing)** via the `@fastly/expressly` framework, running on Fastly Compute (edge-side WebAssembly). The service generates synthetic responses at the edge and requires no upstream backends.

No explicit route handlers or endpoint definitions are detectable from the provided directory snapshot (only `src/index.js` is listed but its contents are not included in the snapshot). The README describes the starter kit as demonstrating **routing and middleware** patterns via `@fastly/expressly`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| _Not determinable from code._ | | | | |

Protocol: **HTTP/1.1 or HTTP/2** (as forwarded through Fastly Compute edge runtime).

---

## Event Schemas

_Not determinable from code._

No message broker, event topics, or asynchronous messaging dependencies are present in the extracted facts or dependency manifests.

---

## Input / Output Formats

- **Serialization / Content-Type:** _Not determinable from code._ The `@fastly/expressly` framework supports JSON and plain-text synthetic responses, but no specific content-type handling is evidenced in the provided snapshot.
- **Pagination:** _Not determinable from code._
- **Request/Response envelope:** _Not determinable from code._
- **Runtime target:** The service compiles to WebAssembly (`./bin/main.wasm`) via `js-compute-runtime` and executes within the Fastly Compute edge environment. All request/response handling occurs at the edge with no backend origin required.

---

## Error Handling

_Not determinable from code._

The source file `src/index.js` is not included in the snapshot. `@fastly/expressly` provides middleware-based error handling conventions, but no specific error-to-response mappings, status codes, or validation behaviours are evidenced in the available materials.

---

## Versioning

_Not determinable from code._

No URI versioning prefix, version headers, or schema evolution strategy is evidenced in the README, `package.json`, or directory structure. The `@fastly/expressly` dependency is pinned to `^2.0.0` and `@fastly/js-compute` to `^3.33.2`, indicating semantic versioning is applied to dependencies, but no API versioning strategy for the service's own interface is detectable.
