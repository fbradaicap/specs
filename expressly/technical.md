---
repo: expressly
spec_type: technical
commit: d15cb937392edb46d2e39d406240dc6fdbf78c6b
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 4e4d8e267b36915bdf0c4034e74b77d1ec77facdcabde45d5d705636dbf3fd75
generated_at: 2026-06-30T14:53:20.999053512+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| **Language** | TypeScript (library: `^5.3.3`; docs: `^5.2.2`) |
| **Runtime target** | Fastly Compute (edge WebAssembly/JS runtime via `@fastly/js-compute ^3.7.0`); Node.js `>=18` required for build tooling |
| **Core framework** | None — this service *is* the framework (`@fastly/expressly v2.4.0`) |
| **Router** | `path-to-regexp ^6.2.1` |
| **Event bus** | `mitt ^3.0.1` (lightweight event emitter) |
| **Cookie parsing** | `cookie ^1.1.1` |
| **Polyfills** | `core-js ^3.35.0` |
| **Build** | `tsc` (TypeScript compiler, emits to `dist/`) |
| **Type testing** | `tsd ^0.30.3` |
| **Formatting** | `prettier ^3.1.1` + `pretty-quick ^4.2.2` |
| **Git hooks** | `husky ^8.0.3` |
| **Documentation site** | Docusaurus `^3.0.0-beta.0` (React 18, deployed separately under `docs/`) |

---

## Architecture Patterns

**This repository is an npm library, not a running microservice.** It provides an Express-style routing framework for applications that run on Fastly Compute (edge/Wasm environment).

Key patterns within the library:

- **Middleware / Handler chain (layered pipeline):** Incoming `FetchEvent` objects are processed through an ordered `requestHandlers` stack; uncaught errors are forwarded to an `errorHandlers` stack. Both stacks are iterated sequentially until the response is ended (`res.hasEnded`).
- **Facade / Express-style API:** `Router` exposes convenience methods (`get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `purge`, `all`, `use`, `route`) that map HTTP methods and path patterns to `RequestHandlerCallback<Req, Res>` instances — mirroring the Express.js API surface.
- **Strategy pattern for matching:** Route matching is delegated to `path-to-regexp`-backed `routeMatcher` functions cached in a `Map<string, Function>` (`pathMatcherCache`) to avoid repeated compilation.
- **Generic type parameters:** `Router<Req, Res, Err>` is fully generic, allowing consumers to extend `EReq`/`ERes`/`EErr` with domain-specific interfaces (demonstrated in `test-d/`).
- **Built-in CORS preflight:** Optional `autoCorsPreflight` configuration automatically registers an `OPTIONS *` handler that validates trusted origins and echoes `Access-Control-Request-Method` / `Access-Control-Request-Headers`.
- **Error boundary:** A default error handler normalises `ErrorNotFound` → 404, `ErrorMethodNotAllowed` → 405 (with `Allow` header), and generic errors → 500 JSON.
- **Event emission:** `mitt` is used internally on the response object; the `finish` event fires with the final `Response | undefined` value (visible in type tests).

Key internal modules under `src/lib/routing/`:

| Module | Role |
|---|---|
| `router.ts` | Central `Router` class; listen, dispatch, middleware/error stacks |
| `request-handler.ts` | Wraps a match function and async callback |
| `error-middleware.ts` | Wraps error-arity (3-arg) callbacks |
| `request.ts` (`ERequest`/`EReq`) | Request abstraction over Fastly `FetchEvent` |
| `response.ts` (`EResponse`/`ERes`) | Response builder with event emission |
| `errors.ts` | `ErrorNotFound`, `ErrorMethodNotAllowed`, `EErr` |

---

## Database & Data Ownership

This service owns **no data stores**. No migrations, ORM models, or database clients are present. The library operates statelessly within the lifetime of a single edge request.

---

## Dependencies

### Runtime dependencies (shipped in `dist/`, required by consumers)

| Package | Version | Purpose |
|---|---|---|
| `@fastly/js-compute` | `^3.7.0` | Fastly Compute JS SDK — provides `FetchEvent`, `addEventListener`, and Wasm platform primitives |
| `path-to-regexp` | `^6.2.1` | Compiles URL patterns to match functions |
| `cookie` | `^1.1.1` | Cookie header parsing |
| `mitt` | `^3.0.1` | Tiny event emitter used on the response object |
| `core-js` | `^3.35.0` | ES polyfills for the Compute runtime |

### Build / development dependencies (not shipped)

| Package | Version | Purpose |
|---|---|---|
| `typescript` | `^5.3.3` | Compilation |
| `tsd` | `^0.30.3` | Type-level test assertions |
| `prettier` | `^3.1.1` | Code formatting |
| `pretty-quick` | `^4.2.2` | Runs Prettier on staged files |
| `husky` | `^8.0.3` | Git pre-commit hook runner |

### Documentation site dependencies (under `docs/`, not part of the library)

`@docusaurus/core`, `@docusaurus/preset-classic`, `react`, `react-dom`, `clsx`, `prism-react-renderer` (all dev/site-build only).

### External service / broker dependencies

_Not determinable from code._ (The library itself makes no outbound service calls; consumers may use `@fastly/js-compute` backends, but those are application-level concerns.)

---

## Deployment Model

**The library itself is not deployed as a service.** It is published to npm as `@fastly/expressly` (scoped to `@fastly`). The deployment artefact is the compiled `dist/` directory produced by `tsc`.

| Aspect | Detail |
|---|---|
| **Published package** | `@fastly/expressly v2.4.0`, `dist/**/*.js`, `dist/**/*.js.map`, `dist/**/*.d.ts` |
| **Build command** | `npm run build` → `tsc` |
| **Pre-publish gate** | `prepack` runs `npm run build && npm run test` (type tests via `tsd`) |
| **Container / Dockerfile** | _Not determinable from code._ — none present in the repository |
| **Kubernetes / Helm / Compose** | _Not determinable from code._ — none present in the repository |
| **Ports** | None — library code; no HTTP server is started by the package itself |
| **Environment variables** | None defined by the library; configuration is passed programmatically via `EConfig` at `Router` instantiation |
| **Health / readiness endpoints** | Not applicable — library, not a running service |
| **Documentation site deployment** | The `docs/` Docusaurus site is built with `yarn build` and served at `https://expressly.edgecompute.app`; CI/CD pipeline details are _not determinable from code._ |
