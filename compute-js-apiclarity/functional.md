---
repo: compute-js-apiclarity
spec_type: functional
commit: ec477c66d41584893a6c511130717e3b262d0b20
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 17babbb596d66b4f9491ef1a5c7159c424601496251836ebe5a3aebc5a0a5872
generated_at: 2026-06-30T14:54:24.588644386+02:00
generator: specsync
---

## Business Purpose

This service runs on the Fastly Compute edge (compiled to WebAssembly via `@fastly/js-compute`) and acts as a bridge between Fastly Next-Gen WAF (NGWAF) sampled log data and a locally running [APIClarity](https://github.com/openclarity/apiclarity) instance. It retrieves a time-windowed batch of NGWAF sampled request/response logs, transforms them into the format expected by APIClarity, and forwards them so that APIClarity can catalogue API traffic and generate OpenAPI specifications automatically.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** API observability / traffic telemetry ingestion.
- **Core entities/aggregates:**
  - NGWAF Sampled Log (retrieved from the NGWAF API; contains request and response data)
  - APIClarity Trace (the formatted payload sent to APIClarity's trace-source endpoint)
- **Neighbouring contexts:**
  - **Upstream:** Fastly Next-Gen WAF — provides the raw sampled log data via its API (authenticated with `NGWAF_EMAIL`, `NGWAF_TOKEN`, `NGWAF_CORP`, `NGWAF_SITE`).
  - **Downstream:** APIClarity — consumes the formatted trace data and exposes a UI for API documentation/spec generation; communication is authenticated via a `TRACE-SOURCE-TOKEN`.

## Use Cases / User Stories

- **As an API platform engineer**, I want to query a time-bounded window of NGWAF sampled logs so that I can capture real API traffic observed by the WAF.
  - Triggered by an HTTP request to the Fastly Compute environment at `/get_sampled_logs` (per README `curl`/`http` example), supplying `NGWAF_EMAIL`, `NGWAF_TOKEN`, `corpName`, `siteName`, and `TRACE-SOURCE-TOKEN` as headers.
- **As an API platform engineer**, I want the raw NGWAF log entries automatically reformatted into the APIClarity trace schema so that no manual data transformation is required.
- **As an API platform engineer**, I want the formatted trace data forwarded to a locally running APIClarity instance so that APIClarity can build an inventory of API endpoints and their observed behaviour.
- **As an API platform engineer**, I want to use the APIClarity UI (exposed on `https://localhost:8443`) to review discovered API endpoints and generate an OpenAPI specification from real traffic.

## Business Rules

- The service **must** receive all four NGWAF credentials/identifiers (`NGWAF_EMAIL`, `NGWAF_TOKEN`, `corpName`, `siteName`) in the request before it can query the NGWAF API. _(inferred from README required headers)_
- A valid `TRACE-SOURCE-TOKEN` **must** be supplied in the request so that the forwarded data is accepted by the APIClarity trace-source endpoint. _(inferred from README)_
- The NGWAF token (`TRACE-SOURCE-TOKEN`) is obtained out-of-band by registering a trace source with APIClarity prior to invoking this service; the service does not create the token itself. _(inferred)_
- Log retrieval is time-bounded ("a period of time's worth of NGWAF Sampled logs"); the exact window size/configuration is `_Not determinable from code._`
- The service operates as a Fastly Compute (WASM) function, meaning all logic executes at the edge within the constraints of the `@fastly/js-compute` runtime (e.g. no persistent local state, network calls subject to Fastly backend configuration). _(inferred from `package.json` build target and dependency)_
- The APIClarity trace-source type used in the demo is `APIGEE_X`; whether other source types are supported is `_Not determinable from code._`
