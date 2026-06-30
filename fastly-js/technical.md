---
repo: fastly-js
spec_type: technical
commit: 2ff10a7eb745a90f2f38e13fe4aaa35ca962115e
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 6ba2660cb9f99488ebf2c1aac9e17f0bd432d3a66d76e4928f652363aa6e4357
generated_at: 2026-06-30T14:53:35.703318254+02:00
generator: specsync
---

## Tech Stack

- **Language:** JavaScript (ES2015+ with Babel transpilation)
- **Runtime target:** Node.js and browser (dual-target; `package.json` sets `"browser": { "fs": false }`)
- **Package version:** 15.1.0
- **Build toolchain:** Babel 7 (`@babel/cli` ^7.16.0, `@babel/core` ^7.20.12) with `@babel/preset-env` and a broad set of proposal plugins (class properties, decorators, optional chaining, nullish coalescing, pipeline operator, etc.)
- **Module format:** CommonJS (compiled output at `dist/index.js`)
- **Runtime HTTP library:** `superagent` ^6.1.0
- **Code generation:** All API and model source files are auto-generated from an OpenAPI 1.0.0 document (noted in every file header)

## Architecture Patterns

This is an **auto-generated OpenAPI client library** (SDK), not a running microservice. Its architecture follows the **API Client / SDK pattern**:

- **`ApiClient` core:** A central `ApiClient` class (in `src/ApiClient.js`) handles authentication (bearer token via `FASTLY_API_TOKEN` environment variable), HTTP transport via `superagent`, request serialisation, and response deserialisation.
- **Per-resource API classes:** Each Fastly API resource domain is encapsulated in a dedicated class under `src/api/` (e.g. `AclApi`, `AclEntryApi`, `BillingAddressApi`, `BillingInvoicesApi`, `CacheSettingsApi`, `ConfigStoreApi`, etc.). Each class exposes both a raw `*WithHttpInfo` variant (returning the full HTTP response) and a convenience wrapper returning only the response data.
- **Model layer:** Data transfer objects are defined under `src/model/` and imported by API classes; these represent request/response schemas.
- **No inbound endpoints, no workers, no event consumers:** The library exposes no server-side endpoints; it is purely an outbound HTTP client library intended to be embedded in consumer applications.
- **Build pipeline:** Source in `src/` is transpiled via Babel to `dist/` before publication; the published package ships only `dist/` and `sig.json`.

## Database & Data Ownership

This service owns no datastore. It is a client library with no database connections, migrations, or persistent storage of any kind.

## Dependencies

### Runtime
| Dependency | Purpose |
|---|---|
| `superagent` ^6.1.0 | HTTP client used by `ApiClient` to make outbound REST calls to `https://api.fastly.com` |

### Build / Dev
| Dependency | Purpose |
|---|---|
| `@babel/cli` ^7.16.0 | CLI tool to invoke Babel transpilation |
| `@babel/core` ^7.20.12 | Babel transpiler core |
| `@babel/preset-env` ^7.16.4 | Transpile modern JS to broad-compatibility ES5/CommonJS |
| `@babel/plugin-proposal-*` (×15) | Enable experimental/proposal syntax (class properties, decorators, optional chaining, nullish coalescing, pipeline operator, etc.) |
| `@babel/register` ^7.16.0 | On-the-fly Babel compilation for Node.js (used during development/testing) |

### External service called at runtime
- **Fastly REST API** at `https://api.fastly.com` — authenticated via a bearer token (`FASTLY_API_TOKEN` env var). The library wraps the full Fastly management API surface: ACLs, backends, billing, cache settings, conditions, config stores, compute resources, domains, and many more resource types.

## Deployment Model

This repository is a **publishable npm package**, not a deployable service. There is no Dockerfile, Kubernetes manifest, Helm chart, or Compose file present.

- **Build:** `npm run build` invokes `babel src -d dist`, writing transpiled output to `dist/`.
- **Packaging:** `npm pack` / `npm publish` triggers `prepack` → `npm run build`; only `dist/` and `sig.json` are included in the published artifact (per `"files"` in `package.json`).
- **Distribution:** Published to npm under the package name `fastly` at version `15.1.0`.
- **Consumer configuration:** Callers must supply the `FASTLY_API_TOKEN` environment variable; the `ApiClient` picks it up automatically in Node.js environments.
- **Ports / health endpoints / container orchestration:** _Not determinable from code._ (None exist; this is a library, not a server.)
