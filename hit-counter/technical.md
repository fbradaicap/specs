---
repo: hit-counter
spec_type: technical
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:44.326295242+02:00
generator: specsync
---

## Tech Stack

- **Language:** JavaScript (Node.js-compatible toolchain, targeting WebAssembly)
- **Runtime:** [Fastly Compute](https://developer.fastly.com/learning/compute/) — JavaScript executed at the edge via `@fastly/js-compute` (^3.30.1), compiled to WebAssembly (WASM)
- **Framework:** `@fastly/expressly` (^2.3.0) — Express-style routing layer for Fastly Compute
- **Notable Libraries:**
  - `lodash` (^4.17.21) — general-purpose utility library (dev dependency)
  - `fastly` (^7.10.0) — Fastly API client (dev dependency, used for local tooling/publish)
  - `@fastly/cli` (^10.14.0) — Fastly CLI for build, local dev, and deployment (dev dependency)

## Architecture Patterns

- **Edge Computing / Serverless at the Edge:** The service runs as a Fastly Compute application, meaning request handling logic executes at Fastly's globally distributed edge PoPs rather than in a centralised data centre.
- **Reverse Proxy + Request Enrichment:** Incoming requests are proxied to an upstream origin (`fastly.github.io` by default); the edge layer intercepts requests to increment hit counters before forwarding, and intercepts `/stats` routes to return synthetic responses.
- **Routing via Middleware Framework:** `@fastly/expressly` provides Express-style route registration, giving the single entry-point (`src/index.js`) a layered request-handling structure.
- **Edge-Side Data Store (KV Store):** A Fastly KV Store named `pagehits` is used to persist and retrieve page hit counts. Reads and writes are performed at the edge without a traditional backend database.
- **Synthetic Response Generation:** The `/stats` route constructs and returns an HTML response entirely at the edge, without hitting the origin.

## Database & Data Ownership

- **Datastore:** Fastly KV Store (managed, key-value, edge-native — not a traditional relational or document database)
- **Store Name:** `pagehits`
- **Ownership:** This service owns the `pagehits` KV Store. Each key corresponds to a URL path; the associated value is the cumulative hit count for that page.
- **Schema:** No formal schema; keys are page path strings, values are integer hit counts.
- No relational DB tables or ORM models are present.

## Dependencies

### Runtime Dependencies
| Dependency | Type | Purpose |
|---|---|---|
| `@fastly/js-compute` (^3.30.1) | Runtime | Core Fastly Compute JavaScript SDK; provides access to KV Store, backends, request/response APIs |
| `@fastly/expressly` (^2.3.0) | Runtime | Edge-compatible Express-style router |

### Build / Development Dependencies
| Dependency | Type | Purpose |
|---|---|---|
| `@fastly/cli` (^10.14.0) | Build/Dev | Fastly CLI; local dev server (`fastly compute serve`) and deployment (`fastly compute publish`) |
| `fastly` (^7.10.0) | Build/Dev | Fastly API client; used by the CLI and tooling during publish |
| `lodash` (^4.17.21) | Build/Dev | General utility functions (used during build/bundle; classified as devDependency in `package.json`) |

### External Service Dependencies
| Service | Purpose |
|---|---|
| `fastly.github.io` (default backend, named `blog`) | Origin website proxied by the edge application; configurable via `fastly.toml` |
| Fastly KV Store (`pagehits`) | Edge-native key-value store for persisting hit count data |

## Deployment Model

- **Build Output:** The `build` script compiles `src/index.js` to a WebAssembly binary at `bin/main.wasm` using `js-compute-runtime`:
  ```
  js-compute-runtime src/index.js bin/main.wasm
  ```
- **Deployment Target:** Fastly Compute platform (edge, not a container or Kubernetes cluster). There is no Dockerfile or Kubernetes/Helm configuration.
- **Deployment Command:** `fastly compute publish` — packages the WASM binary and uploads it to a Fastly Compute service via the Fastly API.
- **Local Development:** `fastly compute serve` runs a local emulator of the Compute runtime.
- **Configuration File:** `fastly.toml` — defines the Compute service, the `blog` backend address (`fastly.github.io`), and the `pagehits` KV Store setup under `[setup]`.
- **Ports:** _Not determinable from code._ — port binding is managed entirely by the Fastly edge platform.
- **Environment Configuration:**
  - `FASTLY_API_TOKEN` environment variable required for deployment/CLI authentication.
  - Backend URL and site root path (`root` variable in `src/index.js`) must be configured before first deployment.
- **Health/Readiness Endpoints:** _Not determinable from code._ — health checks are not applicable in the Fastly Compute execution model; availability is managed by the Fastly platform.
