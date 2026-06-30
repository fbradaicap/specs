---
repo: compute-starter-kit-javascript-expressly
spec_type: technical
commit: d0945ec18eb2d221a3d3da811ba45d7f46dfd215
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 07c150c73a3ef746e559192e517f0af4480943c215c8af3ad7b45210ea33efec
generated_at: 2026-06-30T14:51:30.820608466+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| **Language** | JavaScript (ES Modules — `"type": "module"`) |
| **Runtime** | Fastly Compute (WebAssembly edge runtime via `@fastly/js-compute` ^3.33.2) |
| **Routing framework** | `@fastly/expressly` ^2.0.0 — lightweight Express-style routing layer for Fastly Compute |
| **Build toolchain** | `js-compute-runtime` CLI (bundled with `@fastly/js-compute`) — compiles JS to a WebAssembly binary |
| **Dev/deploy CLI** | `@fastly/cli` ^15.0.0 (dev dependency) |

## Architecture Patterns

- **Edge-compute / serverless at the CDN edge**: The service runs entirely on the Fastly Compute platform, not on a traditional server or container. Execution is triggered per HTTP request at the edge.
- **Single-file entry point**: All logic originates from `src/index.js`, compiled to `bin/main.wasm` at build time.
- **Request-handler / middleware pattern**: Routing and middleware are provided by `@fastly/expressly`, following an Express-style `router.get|post|use(path, handler)` pattern.
- **Synthetic response generation**: No upstream backends are declared; the service constructs and returns responses entirely at the edge without proxying to an origin.
- **Stateless**: No persistent state, database, or event bus — each request is handled independently within the WebAssembly sandbox.

## Database & Data Ownership

This service owns no datastores, tables, or models. It generates all responses synthetically at the edge without reading from or writing to any external or embedded database.

## Dependencies

### Runtime dependencies
| Package | Purpose |
|---|---|
| `@fastly/expressly` ^2.0.0 | Express-style HTTP routing and middleware for Fastly Compute |
| `@fastly/js-compute` ^3.33.2 | Fastly Compute JavaScript SDK and runtime bindings (provides `js-compute-runtime` compiler) |

### Build / development dependencies
| Package | Purpose |
|---|---|
| `@fastly/cli` ^15.0.0 | Fastly CLI — used for local `serve` (development server) and `publish` (deployment) |

### External service dependencies
_Not determinable from code._ (README states no backends are required; no backend declarations are visible in the provided files.)

## Deployment Model

**Build process**
```
npm run build
# → js-compute-runtime ./src/index.js ./bin/main.wasm
```
The JavaScript source is compiled to a self-contained WebAssembly binary (`bin/main.wasm`) using `js-compute-runtime`.

**Deployment target**
The Wasm binary is deployed to the **Fastly Compute** edge network (not a container or Kubernetes cluster) via:
```
npm run deploy
# → fastly compute publish
```
On first deployment, the Fastly CLI creates a new Fastly service in the operator's account.

**Local development**
```
npm run start
# → fastly compute serve
```
Runs a local emulator of the Compute environment.

**Container / orchestration**: _Not determinable from code._ There is no Dockerfile, Kubernetes manifest, or Compose file present; the deployment target is the Fastly platform, not a self-hosted container runtime.

**Ports**: _Not determinable from code._ Port binding is managed by the Fastly platform and local CLI emulator.

**Environment configuration**: _Not determinable from code._ No `.env`, config files, or environment variable references are present in the provided snapshot.

**Health / readiness endpoints**: None — the Fastly Compute platform does not use traditional health-check probes; availability is managed by the platform itself.
