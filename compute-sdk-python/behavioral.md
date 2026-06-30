---
repo: compute-sdk-python
spec_type: behavioral
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:33.154103392+02:00
generator: specsync
---

## API Contracts

This microservice is a **Python SDK for Fastly Compute** (WebAssembly edge runtime), not a standalone HTTP service. Its "endpoints" are defined in example WSGI applications bundled with the SDK. The protocol is **HTTP/1.1** routed through a WSGI adapter (`fastly_compute.wsgi.WsgiHttpIncoming`).

Two example applications (Bottle and Flask) expose identical route sets:

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| ANY | `/hello/<name>` | Returns a plain-text greeting using the URL parameter `name` | Path parameter: `name` (string) | Plain text: `Hello {name}!` |
| ANY | `/info` | Returns runtime diagnostic information as JSON | None | JSON object (see below) |
| ANY | `/error` | Intentionally raises a `RuntimeError` to exercise error-handling paths | None | Framework-dependent error response |

**`/info` response shape (Bottle variant):**

```json
{
  "service": "fastly-compute-python",
  "status": "ok",
  "message": "Hello from Fastly Compute!",
  "vcpu_time_ms": <integer>,
  "request_method": "<string>",
  "path_info": "<string>",
  "request_headers": { "<header-name>": "<header-value>" }
}
```

**`/info` response shape (Flask variant):**

```json
{
  "service": "fastly-compute-python-flask",
  "status": "ok",
  "message": "Hello from Fastly Compute with Flask!",
  "vcpu_time_ms": <integer>,
  "request_method": "<string>",
  "path_info": "<string>",
  "python_version": "<string>",
  "request_headers": { "<header-name>": "<header-value>" }
}
```

The `REQUEST_METHOD` and `PATH_INFO` values surfaced in `/info` responses are read from the WSGI environ directly, as evidenced by `request.environ.get("REQUEST_METHOD")` and `request.environ.get("PATH_INFO")` in both example apps.

The SDK's **testing interface** (`fastly_compute.testing.ViceroyTestBase`) exposes the following helper methods for test consumers:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `get` | `self.get(path, **kwargs)` | Issues an HTTP GET against the Viceroy-served WASM |
| `post` | `self.post(path, **kwargs)` | Issues an HTTP POST |
| `request` | `self.request(method, path, **kwargs)` | Issues any HTTP method |

Configuration class attributes: `REQUEST_TIMEOUT` (default: 10 s), `WASM_FILE` (default: `"app.wasm"`).

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Serialization:** JSON for the `/info` endpoint (returned as a Python `dict` by the framework, serialized to `application/json` by Bottle/Flask automatically). Plain text (`text/plain`) for `/hello/<name>`.
- **Content negotiation:** Delegated entirely to the Bottle or Flask WSGI framework; no explicit `Content-Type` negotiation is implemented in the SDK layer.
- **Pagination:** _Not determinable from code._
- **Request envelope:** Standard HTTP — no custom envelope or envelope fields are defined.
- **Response envelope:** None for `/hello/<name>` and `/error`. For `/info`, the JSON object is a flat dictionary with the fields documented above; there is no outer wrapper envelope.
- **Runtime bindings:** The `vcpu_time_ms` field is populated via the WIT-bound import `compute_runtime.get_vcpu_ms()`, which returns an integer number of milliseconds of vCPU time consumed.

---

## Error Handling

**Application-level (`/error` endpoint):**  
Both example apps raise `RuntimeError("This is an intentional error for testing purposes")` unconditionally. The resulting HTTP status code is determined by the WSGI framework (Bottle/Flask default unhandled-exception behaviour — typically `500 Internal Server Error`); the exact response body is _not determinable from code_.

**SDK exception model (`fastly_compute.exceptions`):**  
The SDK defines a hierarchy rooted at `FastlyError` for errors originating from Fastly WIT bindings:

| Exception class | Trigger |
|---|---|
| `FastlyError` | Base class for all Fastly API errors |
| `UnexpectedFastlyError` | Raised when a WIT `Err` value carries a type not present in the provided mapping; preserves the original `.value` |
| Subclasses (e.g., `BufferTooShortError`, `NegativeHeightError`, `InvalidSyntaxError`, `NotFoundError`) | Raised by `remap_wit_errors` when the `Err` value type matches a declared mapping |

**`remap_wit_errors` decorator behaviour:**

- Wraps callables that may raise `componentize_py_types.Err`.
- Accepts a mapping of `{ErrorType: ExceptionClass}` (or enum member → exception class).
- On `Err(value=V)`: looks up `type(V)` (or `V` directly for enum members) in the mapping; instantiates the corresponding exception with `V` as its sole constructor argument (for typed errors) or with no arguments (for enum-member matches).
- If the type is not found in the mapping, raises `UnexpectedFastlyError` preserving `.value`.

**No HTTP-level custom error payload structure** is defined by the SDK itself; error serialization is delegated to the host WSGI framework.

---

## Versioning

- The SDK package itself is versioned via `pyproject.toml`: **`version = "0.1.2"`**.
- The Rust extension crate (`fastly-compute-py`) mirrors this at **`version = "0.1.2"`** in `crates/fastly-compute-py/Cargo.toml`.
- **No URI versioning, `Accept`-header versioning, or schema evolution strategy** is implemented in the example applications or the SDK's HTTP layer.
- The Python ABI is pinned to **CPython 3.12+ (abi3-py312)** via maturin/PyO3 configuration, producing a single stable-ABI wheel (`cp312-abi3-*`) compatible with all future CPython versions without rebuilding.
