---
repo: compute-js-apiclarity
spec_type: technical
commit: ec477c66d41584893a6c511130717e3b262d0b20
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 17babbb596d66b4f9491ef1a5c7159c424601496251836ebe5a3aebc5a0a5872
generated_at: 2026-06-30T14:54:24.588644386+02:00
generator: specsync
---

## Tech Stack

- **Language:** JavaScript (ES Modules — `"type": "module"`)
- **Runtime:** Fastly Compute (WebAssembly/WASI runtime via `@fastly/js-compute` ^3.7.0)
- **Framework:** `@fastly/expressly` ^2.2.1 — Express-like routing layer for Fastly Compute
- **Build tooling:** `js-compute-runtime` (bundled with `@fastly/js-compute`) compiles the JS entry point to a `.wasm` binary; Fastly CLI (`fastly compute publish`) handles deployment

## Architecture Patterns

- **Edge Compute / Serverless at the edge:** The service runs as a WebAssembly module on the Fastly Compute platform, not as a traditional long-running server process.
- **Request-driven integration proxy:** On each inbound HTTP request the service acts as a pipeline adapter:
  1. Receives a trigger request (with credentials and configuration as headers).
  2. Calls the Fastly Next-Gen WAF (NGWAF) API to retrieve sampled logs for a time window.
  3. Transforms the NGWAF log records into the APIClarity trace format.
  4. Forwards the transformed data to a locally running APIClarity instance.
- **Single entry-point handler:** One source file (`src/index.js`) wired through the Expressly router; the README documents a single logical route (`/get_sampled_logs`).
- No background workers, queues, or persistent state — purely request/response.

## Database & Data Ownership

This service owns no datastore and defines no schema or models. All data is sourced transiently from the NGWAF API at request time and forwarded to APIClarity; nothing is persisted by the service itself.

## Dependencies

### Runtime dependencies
| Dependency | Version | Role |
|---|---|---|
| `@fastly/js-compute` | ^3.7.0 | Fastly Compute JS SDK and `js-compute-runtime` build tool; provides the Fetch-compatible APIs and WASM packaging |
| `@fastly/expressly` | ^2.2.1 | Lightweight Express-style request router for Fastly Compute handlers |

### External service dependencies (runtime, configured via request headers/env)
| Dependency | Purpose |
|---|---|
| **Fastly NGWAF API** (`https://docs.fastly.com/en/ngwaf/extract-your-data`) | Source of sampled WAF log data; authenticated via `NGWAF_EMAIL` + `NGWAF_TOKEN` credentials, scoped to `NGWAF_CORP` / `NGWAF_SITE` |
| **APIClarity** (local, `https://localhost:8443`) | Destination for formatted trace data; authenticated via `TRACE-SOURCE-TOKEN` header |

### Build / developer tooling (not shipped in the wasm binary)
- Fastly CLI — packaging and deployment (`fastly compute publish`)
- `kubectl` — managing the local APIClarity Kubernetes deployment
- Docker CLI — building/running the APIClarity container image
- `npm` — dependency management
- `make` — orchestrates build and demo targets

## Deployment Model

- **Build output:** `npm run build` invokes `js-compute-runtime ./src/index.js ./bin/main.wasm`, producing a self-contained WebAssembly binary at `bin/main.wasm`.
- **Deployment:** `npm run deploy` (alias: `fastly compute publish`) uploads the `.wasm` package to the Fastly Compute edge network via the Fastly CLI. There is no Dockerfile, Kubernetes manifest, or Compose file for the Compute service itself.
- **Local development / testing:** The Fastly CLI provides a local emulator; the README shows the service responding on `http://127.0.0.1:7676`. The companion APIClarity service is run locally via Docker/kubectl and listens on `https://localhost:8443`.
- **Ports:** `7676` (local Fastly Compute emulator); `8443` (local APIClarity HTTPS API). Production port assignment is managed entirely by the Fastly edge platform.
- **Environment configuration:** Required environment variables (`NGWAF_EMAIL`, `NGWAF_TOKEN`, `NGWAF_CORP`, `NGWAF_SITE`) are consumed by the caller and passed as HTTP request headers to the Compute service at invocation time. The `TRACE-SOURCE-TOKEN` is similarly passed as a request header.
- **Health/readiness endpoints:** _Not determinable from code._
