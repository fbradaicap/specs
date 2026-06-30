---
repo: compute-js-apiclarity
spec_type: behavioral
commit: ec477c66d41584893a6c511130717e3b262d0b20
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 17babbb596d66b4f9491ef1a5c7159c424601496251836ebe5a3aebc5a0a5872
generated_at: 2026-06-30T14:54:24.588644386+02:00
generator: specsync
---

## API Contracts

**Protocol:** HTTP (REST-style), served via a Fastly Compute edge runtime using `@fastly/expressly`.

The service exposes a single endpoint inferred from the README usage example. No OpenAPI/proto/GraphQL artifacts are present in the snapshot; the endpoint is evidenced only by the `curl`/`http` invocation in the README.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` (or unspecified) | `/get_sampled_logs` | Query NGWAF sampled logs for a time period, format the data, and forward it to a locally running APIClarity trace ingestion endpoint | Headers: `NGWAF_EMAIL`, `NGWAF_TOKEN`, `TRACE-SOURCE-TOKEN`; Query params (or headers): `corpName`, `siteName` | _Not determinable from code._ |

> **Note:** The HTTP method is not explicitly stated in the README command (`http http://127.0.0.1:7676/get_sampled_logs …` defaults to `GET` in the `httpie` CLI when no body is supplied, but this cannot be confirmed from source code). No additional endpoints are detectable.

---

## Event Schemas

_Not determinable from code._

No message broker, event topic, queue, or asynchronous messaging pattern is referenced in the source snapshot, dependency list, or README.

---

## Input / Output Formats

**Content type / serialization:**
- The APIClarity trace source token is obtained via a JSON POST (`Content-Type: application/json`) to APIClarity's own control API — this is an outbound call made during setup, not served by this service.
- The formatted NGWAF log payload sent to APIClarity is implied to be JSON, consistent with APIClarity's trace ingestion API, but the exact envelope shape is _not determinable from code._

**Request to `/get_sampled_logs`:**
- Credentials and identifiers are passed as HTTP headers (`NGWAF_EMAIL`, `NGWAF_TOKEN`, `corpName`, `siteName`, `TRACE-SOURCE-TOKEN`) based on the README example.
- No request body format or pagination scheme is evidenced.

**Response from `/get_sampled_logs`:**
- _Not determinable from code._

---

## Error Handling

_Not determinable from code._

No source files in `src/` are included in the snapshot (only the directory entry for `src/` is listed with 1 file), so no error-handling logic, status code mappings, validation behaviour, or error payload structures can be determined.

---

## Versioning

_Not determinable from code._

No URI version prefix (e.g., `/v1/`), versioning header, or schema evolution strategy is evidenced in the README, directory tree, or dependency manifests.
