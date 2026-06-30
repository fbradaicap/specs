---
repo: Viceroy
spec_type: behavioral
commit: d7ccf104ed0b5c4399a97a9442279c97705d749a
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ed522038d502957e706c70229783a2ab1afd79135845f526027725ea31a6bed8
generated_at: 2026-06-30T14:52:45.787430157+02:00
generator: specsync
---

## API Contracts

Viceroy is a **local testing daemon / WebAssembly runtime** for Fastly Compute, not a networked microservice with conventional REST, GraphQL, or gRPC endpoints. Its public interface is the **Fastly Compute ABI** — a set of host-call bindings exposed to guest WebAssembly modules via the `witx`/WIT interface definition language and implemented through Wasmtime.

The canonical ABI definitions live in `wasm_abi/compute-at-edge-abi/**/*.witx` and `wasm_abi/wit/**/*`. No HTTP/REST, GraphQL, or gRPC endpoints are exposed by the service itself; the only synchronous interface is the **WebAssembly host-call ABI** consumed by guest `.wasm` binaries at runtime.

| Layer | Protocol | Description |
|---|---|---|
| Guest ↔ Host | WebAssembly host calls (`witx`/WIT) | Fastly Compute platform ABI functions (HTTP, backends, dictionaries, cache, crypto, etc.) |
| CLI invocation | Local process (stdin/args) | `viceroy <wasm-file> [options]` — starts a local Hyper-based HTTP server that forwards requests to the guest Wasm module |
| Inbound HTTP (test server) | HTTP/1.1 via Hyper 0.14 | Accepts arbitrary HTTP requests and dispatches them to the compiled guest Wasm; the path/method/headers are fully pass-through and defined by the guest, not Viceroy |

Because no explicit route table or OpenAPI/proto artifact is present in the snapshot, the specific host-call function signatures are _Not determinable from code_ at this level of snapshot detail.

---

## Event Schemas

_Not determinable from code._

No message broker integration (Kafka, RabbitMQ, etc.), event topics, or asynchronous event publishing/consuming is evidenced in the extracted facts or source samples.

---

## Input / Output Formats

| Aspect | Detail |
|---|---|
| Inbound request format | Arbitrary HTTP/1.1 requests (pass-through to guest); no fixed schema enforced by Viceroy itself |
| Guest Wasm ABI serialization | Low-level WebAssembly linear-memory passing as defined by `witx`/WIT; binary, not JSON or Protobuf |
| Configuration files | TOML (`toml = "^0.5.9"`) — service configuration (backends, dictionaries, etc.) is parsed from a TOML file supplied via CLI |
| JSON | `serde_json` is present; used for structured configuration parsing and test tooling, not for a public API wire format |
| Compression | `flate2` is a dependency; HTTP body decompression for guest requests is supported |
| TLS | `rustls` + `tokio-rustls`; the local test server optionally accepts TLS; certificates supplied via CLI flags or PEM files (`rustls-pemfile`) |
| Content types | Passed through from the inbound HTTP request to the guest; not constrained by Viceroy |
| Pagination | _Not determinable from code._ |

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Guest Wasm trap / fatal error | The `test-fatalerror-config` feature flag (see `cli/tests/trap-test/Cargo.toml`) triggers a fatal-error path in which Wasmtime terminates the guest instance; the host returns an HTTP error response to the caller |
| Wasmtime runtime errors | Propagated as `anyhow::Error` internally; surface as 500-class responses to the HTTP test client |
| Configuration parse errors | TOML/JSON parse failures via `serde`/`toml`; reported to stderr via `tracing` and cause process exit |
| Host-call ABI violations | Handled by `wiggle`-generated glue; invalid guest memory pointers or out-of-range values produce trap errors surfaced through Wasmtime's call-hook mechanism (the `call-hook` feature is enabled) |
| TLS errors | `rustls` errors propagated through `tokio-rustls`; logged and connection closed |
| Error payload structure | _Not determinable from code._ — specific HTTP error body shapes returned to HTTP test clients are defined by guest Wasm, not Viceroy |

---

## Versioning

| Aspect | Detail |
|---|---|
| Library version | `viceroy-lib` and `viceroy` CLI are both at **0.19.1** (as declared in `Cargo.toml` and `cli/Cargo.toml`) |
| ABI versioning | The Compute platform ABI is versioned via the `witx`/WIT definition files in `wasm_abi/`; the component adapter crate (`viceroy-component-adapter`) is pinned at `0.0.0` (unpublished) |
| Versioning strategy | Semantic versioning on crate releases; no URI-based or header-based API versioning is applicable given the ABI-level interface |
| Changelog enforcement | A CI workflow (`.github/workflows/changelog.yml`) enforces that `CHANGELOG.md` is updated on every pull request (unless labelled `skip-changelog`), indicating changelog-driven version tracking |
| ABI evolution | `witx` is explicitly described as experimentally evolving with expected backwards-incompatible changes; the adapter supports `noshift` and `exports`/`library` feature variants for compatibility scenarios |
