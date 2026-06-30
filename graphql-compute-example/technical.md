---
repo: graphql-compute-example
spec_type: technical
commit: acd3472b4ad7c13c9e26622246735fa9ff16eb3f
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 840b244c7e882e1a2625320741063dbcea7474fb6eb0c568a8f321b43f1df774
generated_at: 2026-06-30T14:49:39.187192395+02:00
generator: specsync
---

## Tech Stack

- **Language:** JavaScript (Node.js ≥ 18.0.0, engine constraint)
- **Runtime target:** Fastly Compute (WebAssembly via `@fastly/js-compute` ^3.7.0); the compiled output is a `.wasm` binary executed on the Fastly edge network — not a standard Node.js server process
- **Web/routing framework:** `@fastly/expressly` ^2.2.0 (Express-like router for Fastly Compute)
- **GraphQL execution:** `graphql-helix` ^1.13.0
- **Bundler (build-time):** `webpack` ^5.89.0 + `webpack-cli` ^5.1.4

## Architecture Patterns

- **Edge-compute / serverless at the edge:** The service is deployed as a Fastly Compute application. Request handling runs inside a WebAssembly sandbox on Fastly's global edge network rather than on a traditional long-running server.
- **Request-handling (REST-like handler style):** `@fastly/expressly` provides an Express-style routing layer that dispatches incoming HTTP requests. GraphQL is served over HTTP (typically a single POST/GET endpoint), with `graphql-helix` handling query parsing and execution.
- **Bundled single-entry-point:** webpack bundles `src/index.js` into `bin/index.js`, which is then compiled by `js-compute-runtime` into `bin/main.wasm`. The entire application is a single self-contained Wasm module.
- **Edge caching / origin offload:** Per the README, a stated architectural intent is caching upstream (origin) responses at the edge to reduce latency and origin load.

Key internal components inferred from the source structure:
| Component | Role |
|---|---|
| `src/index.js` | Application entry point; route registration via Expressly |
| `bin/index.js` | Webpack bundle (generated) |
| `bin/main.wasm` | Compiled Wasm binary deployed to Fastly |

## Database & Data Ownership

This service owns **no datastore and no schema**. No migrations, ORM models, or database connections are present.

Data is sourced at runtime by proxying upstream origin requests (e.g. a JSON placeholder–style API for users and posts, as shown in the README examples). The edge service acts purely as a processing and caching layer.

## Dependencies

### Runtime dependencies
| Package | Version | Purpose |
|---|---|---|
| `@fastly/js-compute` | ^3.7.0 | Fastly Compute JS runtime APIs (fetch, cache, request/response primitives for the Wasm environment) |
| `@fastly/expressly` | ^2.2.0 | Express-style HTTP routing for Fastly Compute |
| `graphql-helix` | ^1.13.0 | GraphQL HTTP request parsing, execution, and response formatting |

### Build-time dependencies
| Package | Version | Purpose |
|---|---|---|
| `webpack` | ^5.89.0 | Bundles `src/index.js` into a single `bin/index.js` file |
| `webpack-cli` | ^5.1.4 | CLI driver for webpack |
| `js-compute-runtime` (via `@fastly/js-compute` CLI) | — | Compiles the JS bundle to `bin/main.wasm` |

### External service dependencies
- **Upstream origin API:** The GraphQL resolvers fetch user and post data from a downstream HTTP origin (exact URL _Not determinable from code._). The Fastly service proxies and optionally caches these responses.
- **Fastly platform:** Deployment and execution are fully tied to the Fastly Compute platform (`fastly compute publish`).

No message broker, cache sidecar, or third-party SaaS integrations are detected.

## Deployment Model

### Build process
```
webpack          # bundles src/index.js → bin/index.js  (prebuild)
js-compute-runtime bin/index.js bin/main.wasm  # compiles bundle → Wasm  (build)
fastly compute publish                          # uploads Wasm package to Fastly  (deploy)
```

- **Artifact:** A Fastly Compute package containing `bin/main.wasm`, uploaded to and executed on the Fastly edge network.
- **Container/Dockerfile:** _Not determinable from code._ No `Dockerfile` is present; packaging is handled entirely by the Fastly CLI toolchain.
- **Kubernetes/Helm/Compose:** Not applicable. Deployment target is the Fastly edge platform, not a container orchestration system.
- **Ports:** Not applicable for Fastly Compute; the platform handles TLS termination and HTTP on standard ports (80/443) transparently.
- **Environment configuration:** _Not determinable from code._ Any Fastly service ID, backend origins, or dictionary/secret store bindings would be defined in a `fastly.toml` manifest, which is not present in the provided snapshot.
- **Health/readiness endpoints:** Not applicable. Fastly Compute does not use Kubernetes-style health probes; liveness is managed by the platform.
