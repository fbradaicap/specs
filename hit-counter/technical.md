---
repo: hit-counter
spec_type: technical
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:25.386423828+02:00
generator: specsync
---

## Tech Stack

- **Language:** JavaScript (ES modules targeting the Fastly Compute runtime)
- **Runtime:** [Fastly Compute (JS Compute)](https://js-compute-reference-docs.edgecompute.app/docs/) — WebAssembly-based edge runtime; not Node.js
- **Compiler/Bundler:** `@fastly/js-compute` v3.30.1 — compiles `src/index.js` to a WebAssembly binary (`bin/main.wasm`) via `js-compute-runtime`
- **Web Framework:** `@fastly/expressly` v2.3.0 — Express-style routing library for Fastly Compute
- **Fastly Management SDK:** `fastly` v7.10.0 (dev) — Fastly API client, used for tooling/deployment scripting
- **CLI:** `@fastly/cli` v10.14.0 (dev) — Fastly CLI for local development (`fastly compute serve`) and deployment (`fastly compute publish`)
- **Utility library:** `lodash` v4.17.21 (dev)

## Architecture Patterns

- **Edge Compute / Request-Intercept pattern:** The service runs entirely at the Fastly edge as a WebAssembly Compute application. All logic executes on incoming HTTP requests at the CDN layer — there is no traditional origin server.
- **Reverse-proxy with mutation:** Incoming requests for site pages are proxied upstream to a configured backend (`fastly.github.io` by default); before forwarding, the service increments a hit counter in a KV Store keyed by the request path.
- **Stateful edge with KV Store:** Persistent state (page hit counts) is maintained in a Fastly-managed KV Store named `pagehits`, accessed directly from edge worker code.
- **Routing via Expressly:** URL routing is handled declaratively using the Expressly middleware framework within `src/index.js`. Two logical route groups are evident: pass-through proxy routes (incrementing counters) and a synthetic `/stats` route (reads and renders all KV entries as HTML).
- **Single-file application:** All application logic resides in `src/index.js`; no layered directory structure is used.

## Database & Data Ownership

- **Datastore type:** Fastly KV Store (globally distributed key-value store, managed by Fastly infrastructure)
- **Store name:** `pagehits`
- **Owned data:** Page hit counts keyed by URL path. Each key is a page path; the value is an incrementing integer hit count.
- **Schema:** No relational schema or migrations; schema is implicit — key: `string` (URL path), value: `string` (integer count).
- **Ownership boundary:** The KV Store is provisioned and managed through Fastly's platform (declared in `fastly.toml`). The service has exclusive read/write ownership over the `pagehits` store.

## Dependencies

### Runtime Dependencies
| Dependency | Type | Purpose |
|---|---|---|
| `@fastly/js-compute` v3.30.1 | Runtime | Fastly Compute JS SDK; provides `fetch`, KV Store APIs, and the `js-compute-runtime` compiler |
| `@fastly/expressly` v2.3.0 | Runtime | HTTP routing framework for Fastly Compute |
| **Fastly KV Store** (`pagehits`) | Runtime (platform) | Persistent edge storage for hit counts |
| **Backend origin** (`fastly.github.io`) | Runtime (upstream) | Default proxied origin; configurable in `fastly.toml` |

### Build / Development Dependencies
| Dependency | Type | Purpose |
|---|---|---|
| `@fastly/cli` v10.14.0 | Build/Dev | Local dev server (`serve`) and deployment (`publish`) |
| `fastly` v7.10.0 | Build/Dev | Fastly REST API client (tooling/scripting) |
| `lodash` v4.17.21 | Build/Dev | General-purpose utility functions (used during build or in source) |

## Deployment Model

- **Build output:** `src/index.js` is compiled to `bin/main.wasm` using `js-compute-runtime` (`npm run build`)
- **Deployment target:** Fastly Compute service — the `.wasm` binary is uploaded to Fastly's edge network via `fastly compute publish` (`npm run deploy`); there is no container image or Kubernetes involvement
- **Local development:** `fastly compute serve` (`npm start`) runs a local Compute emulator
- **Configuration:** `fastly.toml` (not shown in full) declares the service, backend addresses (e.g., `[setup.backends.blog]`), and KV Store bindings; the `root` URL path is a variable in `src/index.js`
- **Ports:** _Not determinable from code._ (managed by Fastly's edge infrastructure; standard HTTPS 443 at the edge)
- **Environment variables:** `FASTLY_API_TOKEN` is required for deployment authentication
- **Containerisation/Orchestration:** None — deployment is fully managed by Fastly's platform (no Dockerfile, no Kubernetes manifests)
- **Health/readiness endpoints:** _Not determinable from code._
