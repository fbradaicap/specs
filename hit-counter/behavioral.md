---
repo: hit-counter
spec_type: behavioral
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:25.386423828+02:00
generator: specsync
---

## API Contracts

**Protocol:** HTTP/REST, served at the edge via Fastly Compute (WebAssembly runtime) using the Expressly routing framework.

Two routes are evidenced from the README and application description:

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/{root}/` (e.g. `/my-site/`) and all sub-paths under root | Proxy request to origin backend (`fastly.github.io`) and increment the hit counter for the requested page in the `pagehits` KV Store | Standard HTTP GET; no special request body | Proxied HTML response from origin backend |
| GET | `/{root}/stats/` (e.g. `/my-site/stats/`) | Return a synthetic HTML page listing all tracked pages and their hit counts from the `pagehits` KV Store | No request body | Synthetic HTML page with page hit data |

> Specific field names, exact path parameter patterns, and full handler logic are not fully determinable from the provided snapshot alone; `src/index.js` content was not included. The above is grounded in the README description.

No OpenAPI, GraphQL schema, or proto definition files are present in the snapshot.

---

## Event Schemas

_Not determinable from code._

No message broker, event topics, or pub/sub mechanisms are referenced in the dependency manifest, extracted facts, or README.

---

## Input / Output Formats

- **Content type (proxied routes):** Inherited from the origin backend (`fastly.github.io`); expected to be `text/html` for standard page responses.
- **Content type (`/stats/` route):** Synthetically generated `text/html` response rendered at the edge.
- **Serialization:** No JSON API, Protobuf, or Avro serialization is evidenced. The service operates as an edge proxy with a single synthetic HTML endpoint.
- **KV Store interaction:** Page hit counts are read from and written to a Fastly KV Store named `pagehits`. Keys are page paths; values are numeric hit counts. The exact serialization format of stored values (e.g. plain integer string vs. JSON) is not determinable from the provided snapshot.
- **Pagination:** _Not determinable from code._
- **Request/response envelope:** No custom envelope structure is evidenced; responses are plain HTTP.

---

## Error Handling

_Not determinable from code._

The `src/index.js` source file content was not included in the snapshot. Error handling behaviour (e.g. KV Store miss handling, upstream origin errors, HTTP status codes returned on failure) cannot be determined. Expressly provides framework-level routing error handling, but specific exception-to-response mappings are not evidenced.

---

## Versioning

No API versioning strategy (URI versioning, header-based versioning, or schema evolution mechanism) is evidenced in the snapshot. The URL structure uses a configurable `root` path prefix (defaulting to `/my-site/`) for namespacing, but this is a content routing concern rather than an API versioning strategy.
