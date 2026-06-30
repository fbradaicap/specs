---
repo: hit-counter
spec_type: behavioral
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:44.326295242+02:00
generator: specsync
---

## API Contracts

**Protocol:** HTTP (REST-like routing via Fastly Expressly framework, deployed as a Fastly Compute edge service)

The service intercepts all inbound HTTP requests and routes them based on path. Two logical route categories are evidenced from the README and source description:

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| `GET` | `/<root>/*` (e.g. `/my-site/*`) | Proxy page request to origin backend (`fastly.github.io`), increment the hit count for that path in the `pagehits` KV Store, and return the origin response | Standard HTTP request; no special request body | Proxied HTML response from origin backend |
| `GET` | `/<root>/stats/` (e.g. `/my-site/stats/`) | Return a synthetic HTML page listing all tracked pages and their hit counts from the `pagehits` KV Store | None | Synthetic HTML page containing page paths and hit counts |

> **Note:** Static endpoint paths are not declared in a machine-readable API specification (no OpenAPI/proto/GraphQL schema is present in the snapshot). The routes above are derived from README documentation and the described application behaviour. The exact path prefix is configurable via the `root` variable in `src/index.js`.

---

## Event Schemas

_Not determinable from code._

No message broker, event topics, or asynchronous messaging integrations are present or referenced in the extracted facts or dependency manifests.

---

## Input / Output Formats

- **Content types:** HTML (`text/html`) is the evident output format for both the proxied origin responses and the synthetic `/stats/` page. No JSON API responses are evidenced.
- **Serialization:** No structured serialization format (JSON, Protobuf, Avro) is used for end-user responses. KV Store values are stored and retrieved as numeric hit counts (integer strings) using the Fastly KV Store API (`pagehits` store).
- **Pagination:** _Not determinable from code._
- **Request envelope:** No custom request envelope; standard HTTP requests are passed through directly.
- **Response envelope:** No structured response envelope; responses are either proxied origin HTML or synthetically constructed HTML.
- **KV Store interaction:** The `pagehits` KV Store is keyed by page path; each value represents an integer hit count that is read, incremented, and written back on each eligible request.

---

## Error Handling

_Not determinable from code._

No explicit error payload structure, HTTP error status codes, exception-to-response mappings, or validation behaviour are evidenced in the available snapshot. The Fastly Compute runtime and Expressly framework will apply their own default error handling (e.g. `500` for unhandled exceptions at the edge), but no application-level error model is determinable from the provided files.

---

## Versioning

_Not determinable from code._

No URI versioning prefix (e.g. `/v1/`), version request header, or schema evolution strategy is present or referenced in the snapshot. The application exposes a single, unversioned routing surface.
