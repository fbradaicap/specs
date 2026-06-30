---
repo: compute-sdk-python
spec_type: behavioral
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:10.797313958+02:00
generator: specsync
---

## API Contracts

This microservice is a **Python SDK** for Fastly Compute (WebAssembly/WASM edge runtime), not a standalone HTTP service. It provides a framework for building WSGI-based HTTP handlers that run on the Fastly Compute platform. The extracted endpoints are defined within bundled example applications, not the SDK library itself.

**Protocol**: HTTP (via WSGI adapter `fastly_compute.wsgi.WsgiHttpIncoming`)

### Example Application Endpoints

The following endpoints are evidenced in the example applications (`bottle-app` and `flask-app`) included in the repository:

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| ANY | `/hello/<name>` | Returns a plain-text greeting for the given name | Path parameter `name` (string) | `Hello {name}!` (plain text) |
| ANY | `/info` | Returns runtime and request metadata as JSON | None | JSON object (see below) |
| ANY | `/error` | Intentionally raises a `RuntimeError` for error-handling tests | None | Unhandled exception / error response |
| GET | `REQUEST_METHOD` | WSGI environ key access (internal SDK detail) | — | — |
| GET | `PATH_INFO` | WSGI environ key access (internal SDK detail) | — | — |

**`/info` response body (evidenced from source):**

Bottle variant:
```json
{
  "service": "fastly-compute-python",
  "status": "ok",
  "message": "Hello from Fastly Compute!",
  "vcpu_time_ms": <int>,
  "request_method": "<string>",
  "path_info": "<string>",
  "request_headers": { "<header>": "<value>" }
}
```

Flask variant adds `"python_version"` and changes `"service"` to `"fastly-compute-python-flask"` and `"message"` to `"Hello from Fastly Compute with Flask!"`.

### SDK Testing Interface

The SDK exposes a Python test helper (`fastly_compute.testing.ViceroyTestBase`) with the following synchronous HTTP client methods:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `get` | `self.get(path, **kwargs)` | Performs a GET request against the Viceroy-hosted WASM |
| `post` | `self.post(path, **kwargs)` | Performs a POST request |
| `request` | `self.request(method, path, **kwargs)` | Performs any HTTP method request |

Configuration class attributes:

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `REQUEST_TIMEOUT` | `10` (seconds) | HTTP request timeout |
| `WASM_FILE` | `"app.wasm"` | Path to the compiled WASM binary |

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Serialization**: JSON for `/info` endpoint responses (evidenced by dict return values in Flask/Bottle handlers, which both frameworks automatically serialize to `application/json`). Plain text (`text/plain`) for `/hello/<name>`.
- **Content types**: Determined by the WSGI framework in use (Bottle or Flask); no explicit `Content-Type` overrides are evidenced in the SDK source.
- **Request format**: Standard HTTP requests routed through the `WsgiHttpIncoming` adapter, which translates Fastly Compute's WIT-based HTTP interface into a WSGI-compatible `environ` dict.
- **WSGI environ keys used**: `REQUEST_METHOD`, `PATH_INFO` (evidenced in example source).
- **Pagination**: _Not determinable from code._
- **Request/response envelopes**: No envelope wrapper; responses are bare JSON objects or plain strings as returned by the WSGI framework.

---

## Error Handling

### Application-level (`/error` endpoint)
- The `/error` route intentionally raises `RuntimeError("This is an intentional error for testing purposes")`. The resulting HTTP error response format depends on the WSGI framework (Bottle or Flask) and is _not determinable from code_ beyond an unhandled exception producing a 5xx response.

### SDK Exception Remapping (`remap_wit_errors`)
The SDK provides a decorator `fastly_compute.runtime_patching.decorators.remap_wit_errors` that maps WIT `result`-type errors (surfaced as `componentize_py_types.Err`) into Python exceptions:

| Scenario | Behaviour |
|----------|-----------|
| `Err` value type matches a key in the provided mapping dict | Raises the mapped `FastlyError` subclass, passing the `Err` value to its constructor |
| `Err` value is an enum member matching a key in the mapping dict | Raises the mapped `FastlyError` subclass with no constructor arguments |
| `Err` value type is not in the mapping | Raises `UnexpectedFastlyError` wrapping the original value (accessible as `.value`) |

**Exception hierarchy (evidenced):**
- `FastlyError` — base class for all Fastly API errors
  - `UnexpectedFastlyError` — wraps unexpected WIT error values; exposes `.value`
  - User-defined subclasses (e.g., `BufferTooShortError`, `NegativeHeightError`, `InvalidSyntaxError`, `NotFoundError`)

**HTTP status codes**: _Not determinable from code._ Status codes for error responses are delegated to the underlying WSGI framework.

### Test Assertions (evidenced in test suite)
- HTTP `200` is the expected success status for well-formed requests (evidenced by `assert response.status_code == 200` in README).

---

## Versioning

No URI versioning, header-based versioning, or schema evolution strategy is evidenced in the source. The SDK package version is `0.1.2` (from `pyproject.toml` and `crates/fastly-compute-py/Cargo.toml`), but no API version is propagated into endpoint paths or HTTP headers. _Not determinable from code._
