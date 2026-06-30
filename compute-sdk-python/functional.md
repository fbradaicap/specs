---
repo: compute-sdk-python
spec_type: functional
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:10.797313958+02:00
generator: specsync
---

## Business Purpose

`fastly-compute` is a Python SDK that enables developers to write and deploy serverless applications on the Fastly Compute edge platform. It provides WSGI adapter infrastructure (allowing standard Python web frameworks such as Flask and Bottle to run on Fastly's WebAssembly-based edge runtime), a build toolchain for compiling Python applications to WASM components, and a testing harness for running integration tests against a local Viceroy (Fastly's local Compute runtime emulator). It exists to close the gap between standard Python web development practices and Fastly's edge-native, WebAssembly Component Model execution environment.

---

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Fastly Compute edge application development tooling for Python.
- **Core domain entities / aggregates owned:**
  - `WsgiHttpIncoming` – adapter that bridges a WSGI-compatible Python application to the Fastly Compute WIT-based HTTP host interface.
  - `FastlyResource` / `WitResource` – lifecycle-managed wrappers around WIT-bound host resources (e.g., `ConfigStore`, `RateCounter`, `PenaltyBox`, `LogEndpoint`).
  - WIT world bindings (`wit_world`) – generated interface types representing Fastly's host capabilities (HTTP, compute runtime, types, open/config stores).
  - WASM component build artefact – produced by the `fastly-compute-py` Rust crate via `componentize-py`.
  - `ViceroyTestBase` / `AutoViceroyTestBase` – test-time abstractions wrapping a locally managed Viceroy process.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Fastly's WIT interface definitions (found in `wit/`) define the host ABI; the SDK depends on these as its authoritative contract.
  - **Upstream (build):** `componentize-py` (Bytecode Alliance) is used to compile Python + WIT into a WASM component; `wac-graph`/`wac-parser` are used to compose components.
  - **Downstream:** End-user Fastly Compute services written in Python (e.g., the `bottle-app`, `flask-app`, `backend-requests`, `game-of-life` examples) consume this SDK as a library dependency.
  - **Runtime peer:** Viceroy (Fastly's local emulator) is used at test time as the execution environment for compiled WASM components.

---

## Use Cases / User Stories

- **As a Python developer**, I want to wrap a Bottle or Flask WSGI application with `WsgiHttpIncoming` so that it can handle HTTP requests on the Fastly Compute edge without rewriting my application logic.
  - Evidence: `examples/bottle-app/bottle-app.py`, `examples/flask-app/flask-app.py`, `fastly_compute/wsgi` module.
- **As a Python developer**, I want to build my Python application into a Fastly-compatible WASM component using the `fastly-compute-py` CLI (`fastly-compute-py` entry point) so that it can be deployed to Fastly Compute.
  - Evidence: `[project.scripts] fastly-compute-py`, `crates/fastly-compute-py/`, `componentize-py` dependency.
- **As a Python developer**, I want to access Fastly edge capabilities (vCPU time, config stores, rate counters, penalty boxes, log endpoints) from my application code so that I can use platform-native features.
  - Evidence: `fastly_compute/_resource.py` (ConfigStore, RateCounter, PenaltyBox, LogEndpoint mentioned), `wit_world.imports.compute_runtime.get_vcpu_ms()` calls in examples.
- **As a Python developer**, I want to write pytest-based integration tests using `ViceroyTestBase` that automatically start and stop a local Viceroy server so that I can validate my edge application logic without deploying to production.
  - Evidence: README Quick Start, `fastly_compute/testing` module, `fastly_compute/tests/`.
- **As a Python developer**, I want WIT `Err`-typed exceptions automatically remapped to idiomatic Python exceptions via `remap_wit_errors` so that I can handle Fastly API errors using standard Python exception patterns.
  - Evidence: `fastly_compute/tests/test_exception_remapping.py`, `fastly_compute/exceptions.py`, `fastly_compute/runtime_patching/decorators.py`.
- **As a Python developer**, I want to use standard web frameworks (Flask, Bottle) and have their request/response lifecycle transparently adapted to Fastly's HTTP host interface so that I can reuse existing framework knowledge on the edge.
  - Evidence: `/hello/<name>`, `/info`, `/error` routes in both example apps; `WsgiHttpIncoming` adapter.
- **As a CI pipeline**, I want test failures to automatically display recent Viceroy server output via the pytest plugin so that debugging failed edge application tests is straightforward.
  - Evidence: README "Enabling Automatic Viceroy Output", `fastly_compute/pytest_plugin` module.

---

## Business Rules

- **WASM artefact is a prerequisite for testing:** A compiled `.wasm` file (default `app.wasm`, configurable via `WASM_FILE`) must exist before tests are executed; the SDK does not build it automatically at test time. _(README: "WASM file must exist (handled by your build system).")_
- **Python version floor:** The SDK and all applications built with it require CPython ≥ 3.12; the published wheel targets the stable ABI (`abi3-py312`) and is compatible with CPython 3.12 and later without rebuilding. _(pyproject.toml `requires-python`, `features = ["pyo3/abi3-py312"]`.)_
- **WIT resources must be explicitly closed or used as context managers:** After a `FastlyResource` (or any `WitResource`) is closed, any further access results in a trap; the SDK documents this invariant explicitly. _(`fastly_compute/_resource.py`.)_
- **Unexpected WIT `Err` values produce `UnexpectedFastlyError`:** If a WIT `Err` is raised with a type not listed in the `remap_wit_errors` mapping, the SDK wraps it in `UnexpectedFastlyError` (a subclass of `FastlyError`) preserving the original value, rather than propagating the raw `componentize-py` `Err` type. _(`test_exception_remapping.py`, `remap_wit_errors` decorator.)_
- **All Fastly API exceptions are catchable via `FastlyError`:** Both expected (mapped) and unexpected WIT errors subclass `FastlyError`, so callers can catch the entire Fastly error hierarchy with a single `except FastlyError`. _(inferred from class hierarchy in `fastly_compute/exceptions.py` and test evidence.)_
- **WSGI entry point must be named `HttpIncoming`:** The WIT world expects the exported HTTP handler to be bound to the name `HttpIncoming`; both example apps assign `HttpIncoming = WsgiHttpIncoming(app)`. _(inferred from both example files.)_
- **Default HTTP request timeout in tests is 10 seconds**, overridable per test class via `REQUEST_TIMEOUT`. _(README Configuration section.)_
- **`remap_wit_errors` enum-case mappings construct exceptions with no arguments:** When an `Err` value is an enum member, the mapped exception class is instantiated with zero constructor arguments. _(inferred from test assertion: `assert len(e.args) == 0`.)_
- **`remap_wit_errors` non-enum (struct/primitive) mappings pass the WIT error value as the sole constructor argument:** The mapped exception class receives the raw WIT error value as its only `__init__` argument. _(inferred from `BufferTooShortError.__init__(self, wit_error)` and `NegativeHeightError.__init__(self, height)` in tests.)_
