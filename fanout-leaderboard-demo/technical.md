---
repo: fanout-leaderboard-demo
spec_type: technical
commit: a12834febfc6421401e713b919d3a7edaabf81be
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 654db5a3047c6cc5c8633c79339ca38316e560d3fd644fcd765daa17e6033702
generated_at: 2026-06-30T14:56:31.067691311+02:00
generator: specsync
---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js ≥ 22.9.0 (backend); Node.js ≥ 18.0.0 (edge build tooling) |
| Language | JavaScript (ESM, `"type": "module"`) |
| Web Framework | Express 5.x |
| Templating | Nunjucks 3.x |
| Frontend | React 18.x + ReactDOM, bundled via Webpack 5 / webpack-cli |
| Frontend transpilation | Babel (`@babel/preset-env`, `@babel/preset-react`), babel-loader |
| CSS bundling | css-loader, style-loader |
| Database client | sqlite + sqlite3 (SQLite, file-based) |
| Realtime / GRIP | `@fanoutio/grip` 4.x, `@fanoutio/serve-grip` 2.x |
| Edge runtime | Fastly Compute (`@fastly/js-compute` 3.x), compiled to WebAssembly via `js-compute-runtime` |
| Edge tooling | Fastly CLI 12.x |
| Dev utilities | nodemon 3.x, npm-run-all 4.x, odometer 0.4.x (animated counters) |

## Architecture Patterns

**Overall style:** Two-tier edge + origin architecture with realtime push via GRIP.

```
Browser ──► Fastly Compute (edge/src/index.js)
              │  Fanout-capable requests held open
              ▼
          Origin: Express backend (server/)
              │
              ▼
           SQLite DB
```

**Key patterns:**

- **Edge proxy / Fastly Compute application** (`edge/`): A WebAssembly-compiled JS program deployed on Fastly's Compute platform. Its sole responsibility is to intercept incoming requests and hand them off to Fastly Fanout for streaming subscriptions, forwarding all other requests to the Node.js origin.
- **GRIP (Generic Realtime Intermediary Protocol)**: The origin acts as a GRIP backend. Streaming (SSE) connections are delegated to Fanout via `serve-grip` middleware; the origin publishes updates through Fanout to subscribed clients on the `board-{id}` channel rather than maintaining long-lived connections itself.
- **Layered / MVC-style backend** (`server/`): Express router (`api.js`) handles HTTP; DB access is abstracted behind model classes (`Board`, `Player` in `server/db/models/`); Nunjucks handles server-side HTML rendering.
- **Server-Sent Events (SSE)**: `GET /:boardId/` negotiates `text/event-stream` and opens a GRIP hold-stream subscription when the client accepts SSE, enabling live leaderboard updates without polling.
- **Bundled SPA frontend**: React client app compiled by Webpack and served as a static bundle by the Express backend.
- **Parallel dev workflow**: `npm-run-all` runs Webpack watch and nodemon concurrently during development.

## Database & Data Ownership

| Aspect | Detail |
|---|---|
| Datastore | SQLite (file-based, embedded) |
| ORM / client | `sqlite` + `sqlite3` npm packages |
| Owned models | `Board`, `Player` (defined in `server/db/models/index.js`) |
| Schema migrations | _Not determinable from code._ (no migration files present in the directory snapshot) |
| Ownership boundary | The origin Node.js server is the sole owner of the SQLite database. The edge/Fanout layer has no direct database access. |

**`Board`** holds leaderboard definitions (accessed by `boardId`).  
**`Player`** holds player records scoped to a board (accessed by `playerId` + `boardId`), with at minimum `id`, `name`, and `score` fields plus a `scoreAdd()` mutation method.

## Dependencies

### Runtime (origin – `package.json` `dependencies`)

| Package | Purpose |
|---|---|
| `express` ^5.1.0 | HTTP server and routing |
| `nunjucks` ^3.2.4 | Server-side HTML templating |
| `react` / `react-dom` ^18.2.0 | Frontend UI framework |
| `sqlite` ^5.0.1 + `sqlite3` ^5.1.6 | SQLite database access |
| `@fanoutio/grip` ^4.3.1 | GRIP protocol primitives (channel/hold instructions, publishing) |
| `@fanoutio/serve-grip` ^2.0.1 | Express middleware integrating GRIP proxy validation and response helpers |

### Runtime (edge – `edge/package.json` `dependencies`)

| Package | Purpose |
|---|---|
| `@fastly/js-compute` ^3.16.2 | Fastly Compute JS runtime APIs and `js-compute-runtime` compiler |
| `@fastly/cli` ^12.0.0 | Fastly CLI (local dev server `fastly compute serve`, deploy) |

### Build / Dev only (origin – `devDependencies`)

| Package | Purpose |
|---|---|
| `@babel/preset-env`, `@babel/preset-react`, `babel-loader` | Transpile React JSX and modern JS for browser bundle |
| `css-loader`, `style-loader` | Bundle CSS into the Webpack output |
| `webpack-cli` ^5.1.4 | Webpack build driver |
| `nodemon` ^3.0.1 | Auto-restart server on source changes during dev |
| `npm-run-all` ^4.1.5 | Parallel dev script runner |
| `odometer` ^0.4.8 | Animated number counter widget used in the frontend |

### External services / infrastructure

| Dependency | Role |
|---|---|
| Fastly Fanout | Managed GRIP proxy; holds open SSE connections and fans out published messages |
| Fastly Compute platform | Hosts and executes the edge WebAssembly application |
| Google App Engine (production) | Hosts the Node.js origin backend (documented in README; no IaC present in repo) |
| Pushpin (local dev only) | Local GRIP proxy substitute used when developing without a Fastly account |

## Deployment Model

### Origin (Node.js backend)

| Aspect | Detail |
|---|---|
| Dockerfile / container | _Not determinable from code._ (no `Dockerfile` present in snapshot) |
| Orchestration | _Not determinable from code._ (no Kubernetes, Helm, or Compose files present) |
| Production host | Google App Engine (per README) |
| Build step | `NODE_ENV=production webpack` — produces the client bundle |
| Start command | `node --env-file-if-exists=.env server/index.js` |
| Combined start | `npm start` = build then serve |
| Dev start | `npm run dev` — runs Webpack `--watch` and `nodemon` in parallel |
| Port | `3000` (local dev; _production port not determinable from code_) |
| Environment config | Loaded from `.env` file via Node.js `--env-file-if-exists` flag |
| Health/readiness endpoints | _Not determinable from code._ |

### Edge (Fastly Compute application)

| Aspect | Detail |
|---|---|
| Source directory | `edge/` |
| Build output | `edge/bin/main.wasm` (compiled from `edge/src/index.js` by `js-compute-runtime`) |
| Build command | `npm run build` → `js-compute-runtime ./src/index.js ./bin/main.wasm` |
| Local dev | `npm run dev` → `fastly compute serve` (uses Viceroy + Pushpin) |
| Local dev port | `7676` |
| Deploy command | `fastly compute publish` |
| Production URL | `leaderboard-demo.edgecompute.app` |
| Configuration | Fastly service must have Fanout feature enabled and the Node.js origin configured as its backend |
