---
repo: js-serve-grip-expressly
spec_type: technical
commit: 60e0350603e451b5ab32cc8883661b7b3e574c8c
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9232c1c7722e84c090d6d59cc77563f4961711f698c88abd545354e81f5d83f4
generated_at: 2026-06-30T14:52:34.254826147+02:00
generator: specsync
---

## Tech Stack

- **Language:** TypeScript (compiled with `tsc`; TypeScript `^4.7.2`)
- **Runtime target:** Fastly Compute (edge runtime — not Node.js), via `@fastly/expressly` and `@fastly/grip-compute-js`
- **Framework:** `@fastly/expressly` `1.0.0-alpha.1` — an Express-style router for Fastly Compute
- **Core libraries (runtime):**
  - `@fanoutio/grip` `^3.1.0` — GRIP protocol primitives (channels, publisher, WebSocket-over-HTTP)
  - `@fanoutio/serve-grip` `^1.2.0` — middleware layer implementing the GRIP protocol over HTTP
  - `@fastly/grip-compute-js` `^0.1.0` — Fastly Compute-specific GRIP transport adapter
  - `debug` `^4.1.1` — debug-level logging
  - `patch-obj-prop` `^1.0.0` — utility for patching object properties at runtime
- **Build tooling (dev):** `typescript`, `eslint` with `@typescript-eslint/*`, `rimraf`
- **Package manager:** pnpm
- **Output formats:** CommonJS (`build/src/`) and ESM (`build/esm/`)

---

## Architecture Patterns

- **Library / middleware package** — this is not a standalone deployable service; it is an npm package (`@fastly/serve-grip-expressly`) that integrates two existing packages (`@fanoutio/serve-grip` and `@fastly/expressly`) for use by consumer Fastly Compute services.
- **Adapter pattern** — the core design is a pair of adapter classes that bridge the Expressly request/response abstractions (`ERequest`/`EResponse`) to the GRIP-protocol-agnostic interfaces (`IApiRequest`/`IApiResponse`) expected by `@fanoutio/serve-grip`:
  - `ExpresslyApiRequest` — wraps `ERequest`, implementing `IApiRequest<ERequest>` (method, headers, body access)
  - `ExpresslyApiResponse` — wraps `EResponse`, implementing `IApiResponse<EResponse>` (status, end)
  - Both use a `WeakMap` cache to ensure a single adapter instance per underlying request/response object (flyweight / identity-map pattern)
- **Middleware pattern** — the public `ServeGrip` class (re-exported from `@fanoutio/serve-grip`, configured for the Expressly context) is consumed as `router.use(serveGrip)`, following standard Express-style middleware composition
- **Event-driven / streaming** — the package enables GRIP-based long-lived HTTP streaming and WebSocket-over-HTTP (WS-over-HTTP) on the Fastly edge, with publish/subscribe semantics via Fastly Fanout or Pushpin

---

## Database & Data Ownership

This service owns no datastore, schema, or persistent data. It is a stateless middleware/adapter library; all state is held externally by the GRIP publisher (Fastly Fanout or Pushpin).

---

## Dependencies

### Runtime dependencies

| Dependency | Role |
|---|---|
| `@fanoutio/grip` `^3.1.0` | GRIP protocol types, publisher, `WebSocketMessageFormat`, channel primitives |
| `@fanoutio/serve-grip` `^1.2.0` | Core GRIP middleware; provides `ServeGrip`, `IApiRequest`, `IApiResponse` |
| `@fastly/expressly` `1.0.0-alpha.1` | Expressly edge router; provides `ERequest`, `EResponse`, `Router` |
| `@fastly/grip-compute-js` `^0.1.0` | Fastly Compute transport adapter for GRIP publishing |
| `debug` `^4.1.1` | Lightweight debug logging |
| `patch-obj-prop` `^1.0.0` | Runtime property patching utility |

### External runtime services (consumer-configured)

| Service | Purpose |
|---|---|
| **Fastly Fanout** (`fanout.fastly.com`) | GRIP proxy and publisher endpoint for production edge streaming |
| **Pushpin** (optional, local dev) | Open-source GRIP proxy for local `fastly compute serve` development |

### Build/dev dependencies

| Dependency | Role |
|---|---|
| `typescript` `^4.7.2` | TypeScript compiler |
| `eslint` + `@typescript-eslint/*` | Linting |
| `rimraf` `^3.0.2` | Build directory cleanup |
| `@types/debug`, `@types/node` | Type definitions |

---

## Deployment Model

This repository produces a **published npm package** (`@fastly/serve-grip-expressly`), not a deployable container or service.

- **Build:** `pnpm build` — runs ESLint then invokes `tsc` twice (once for CJS via `tsconfig.json`, once for ESM via `tsconfig.esm.json`); output lands in `build/`
- **Publish artefact:** the `build/**/*` directory is included in the npm package; entry points are `build/src/index.js` (CJS) and `build/esm/index.js` (ESM), with types at `build/src/index.d.ts`
- **Container/orchestration:** _Not determinable from code._ — no Dockerfile, Kubernetes manifests, or Compose files are present
- **Ports:** _Not determinable from code._ — not applicable for a library package
- **Environment configuration:** Consumer services must configure a Fastly backend named `grip-publisher` (pointing to `fanout.fastly.com` or a local Pushpin instance) in their `fastly.toml`; no environment variables are defined by this package itself
- **Health/readiness endpoints:** _Not determinable from code._ — not applicable for a library package
