---
repo: graphql-compute-example
spec_type: behavioral
commit: acd3472b4ad7c13c9e26622246735fa9ff16eb3f
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 840b244c7e882e1a2625320741063dbcea7474fb6eb0c568a8f321b43f1df774
generated_at: 2026-06-30T14:49:39.187192395+02:00
generator: specsync
---

## API Contracts

**Protocol:** GraphQL over HTTP (POST), served at the edge via Fastly Compute using `@fastly/expressly` and `graphql-helix`.

No explicit REST endpoints are registered in the extracted facts. The service exposes a GraphQL API; the path is `_Not determinable from code._` (commonly `/graphql` by convention with `graphql-helix`, but not evidenced in the snapshot).

### Queries evidenced in README

| Operation | Type | Purpose | Arguments | Return Shape |
|---|---|---|---|---|
| `users` | Query | Retrieve a list of all users | None evidenced | `id`, `name`, `email`, `address { city }`, `company { name }` |
| `posts` | Query | Retrieve a list of all posts | None evidenced | `id`, `title`, `userId`, `body` |

Mutations, subscriptions, and additional query arguments are `_Not determinable from code._`

---

## Event Schemas

_Not determinable from code._

No message broker, event topics, or asynchronous messaging dependencies are present in the extracted facts or dependency manifests.

---

## Input / Output Formats

- **Content type:** GraphQL requests are standard HTTP with `Content-Type: application/json`. The `graphql-helix` library handles parsing of the GraphQL request envelope, which typically expects:
  ```json
  {
    "query": "...",
    "variables": {},
    "operationName": "..."
  }
  ```
- **Serialization:** JSON (standard GraphQL over HTTP transport).
- **Response envelope:** Standard GraphQL response envelope as defined by the GraphQL specification:
  ```json
  {
    "data": { ... },
    "errors": [ ... ]
  }
  ```
- **Pagination:** _Not determinable from code._
- **Transport runtime:** Compiled to WebAssembly (`bin/main.wasm`) via `@fastly/js-compute` and executed on Fastly Compute edge infrastructure.

---

## Error Handling

- **GraphQL-level errors:** Errors encountered during query execution are returned in the standard GraphQL `errors` array within the response body (HTTP 200), per the GraphQL over HTTP specification as implemented by `graphql-helix`. Individual error objects typically carry a `message` field; additional fields (e.g., `locations`, `path`, `extensions`) are `_Not determinable from code._`
- **HTTP-level error codes and custom error payload structure:** _Not determinable from code._
- **Validation:** GraphQL schema validation is handled by the `graphql-helix` runtime prior to execution; validation failures are surfaced as GraphQL errors in the `errors` array rather than as HTTP error status codes.
- **Exception-to-response mappings:** _Not determinable from code._

---

## Versioning

_Not determinable from code._

No URI versioning prefix (e.g., `/v1/`), version request header, or schema versioning strategy is evidenced in the snapshot. The package version is `graphql-compute-example` (no explicit semver in `package.json`), and no API versioning mechanism is observable from the available source.
