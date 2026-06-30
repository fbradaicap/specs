---
repo: compute-sdk-python
spec_type: functional
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:33.154103392+02:00
generator: specsync
---

## Business Purpose

`fastly-compute` is a Python SDK that enables developers to build and deploy applications on Fastly's Compute (edge compute) platform. It provides WSGI bridge adapters, Fastly-specific runtime APIs (config stores, rate counters, log endpoints, etc.), build tooling to compile Python applications to WebAssembly Components, and a testing harness to run those applications locally under the Viceroy emulator.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Fastly Compute edge application authoring and testing in Python.
- **Core entities / aggregates:**
  - `WsgiHttpIncoming` — bridges a standard Python WSGI application to the Fastly Compute HTTP request/response lifecycle.
  - `FastlyResource` — generic wrapper lifecycle for host resources (ConfigStore, RateCounter, PenaltyBox, LogEndpoint) exposed via WIT bindings.
  - WIT-generated bindings (`wit_world`) — typed interfaces to Fastly host APIs (compute runtime, HTTP types, etc.).
  - WASM Component build pipeline — tooling (`fastly_compute_py_build`) that compiles a Python application into a Wasm Component using `componentize-py`.
- **Relationships to neighbouring contexts:**
  - Upstream: Fastly Compute host platform (provides the WIT interfaces consumed by this SDK).
  - Downstream: end-user Python applications (Bottle, Flask, or plain WSGI) that import and use this SDK.
  - Tooling peer: Viceroy (local Fastly Compute emulator) used exclusively in the test/dev path.

## Use Cases / User Stories

- **As a Python developer**, I want to write a standard WSGI or framework (Flask / Bottle) application and deploy it to Fastly Compute, so that I can run Python business logic at the edge without learning a new request/response model. _(evidenced by `WsgiHttpIncoming`, `examples/flask-app`, `examples/bottle-app`)_
- **As a Python developer**, I want a CLI command (`fastly-compute-py`) that compiles my Python application to a Wasm Component, so that it can be deployed to the Fastly Compute platform. _(evidenced by `[project.scripts]` entry and `fastly_compute_py_build` binary in `crates/fastly-compute-py`)_
- **As a Python developer**, I want to access Fastly-specific runtime primitives (config stores, rate counters, penalty boxes, log endpoints, vCPU time), so that I can leverage Fastly edge features from Python code. _(evidenced by `FastlyResource`, `compute_runtime.get_vcpu_ms()` in examples, WIT imports)_
- **As a test author**, I want a pytest-based test harness (`ViceroyTestBase`, `AutoViceroyTestBase`) that automatically manages a local Viceroy server with dynamic port allocation, so that I can write integration tests for my Compute application without manual server setup. _(evidenced by README, `fastly_compute/testing`)_
- **As a test author**, I want a pytest plugin (`fastly_compute.pytest_plugin`) that automatically surfaces Viceroy server logs on test failure, so that I can debug failures quickly. _(evidenced by README and plugin reference)_
- **As a Python developer**, I want WIT-generated errors remapped to idiomatic Python exceptions (`FastlyError` hierarchy, `remap_wit_errors` decorator), so that error handling in my application code is Pythonic rather than tied to `componentize-py` internals. _(evidenced by `fastly_compute/exceptions.py`, `runtime_patching/decorators.py`, `test_exception_remapping.py`)_
- **As a developer validating the SDK**, I want example applications (`/hello/<name>`, `/info`, `/error` endpoints) exercised against Viceroy in CI, so that SDK correctness across HTTP routing, runtime introspection, and error propagation is verified. _(evidenced by `examples/bottle-app`, `examples/flask-app`, extracted endpoint facts)_

## Business Rules

- A WASM file must be present before tests can execute; the SDK does not build it — that is delegated to the external build system. (inferred from README: "WASM file must exist (handled by your build system)")
- The default WASM file name is `app.wasm`; the default request timeout is 10 seconds; both are overridable per test class via `WASM_FILE` and `REQUEST_TIMEOUT` class attributes. _(evidenced by README configuration section)_
- Python ≥ 3.12 is required; the native extension wheel targets the stable ABI `abi3-cp312` so a single wheel is compatible with CPython 3.12 and all later versions. _(evidenced by `requires-python = ">=3.12"` and `features = ["pyo3/abi3-py312"]` in `pyproject.toml`)_
- All Fastly host resources (`FastlyResource` subclasses) must be explicitly closed or used as context managers; using a resource after closure results in a host trap. _(documented in `_resource.py`)_
- `remap_wit_errors` must map every error type that a WIT binding can raise; unmapped error types are re-raised as `UnexpectedFastlyError`, ensuring callers can always catch the `FastlyError` base class. _(evidenced by `test_unexpected` in `test_exception_remapping.py`)_
- Exception classes mapped from enum-valued WIT errors are constructed with no arguments; exception classes mapped from typed WIT errors receive the WIT error instance as their sole constructor argument. _(evidenced by `test_exception_remapping.py` and docstrings in `remap_wit_errors`)_
- The `wasiless` crate provides minimal/trapping stubs for WASI interfaces, enabling non-WASI Wasm modules to link in environments that require those interfaces without providing real implementations. _(evidenced by `wasiless/Cargo.toml` description)_
- The SDK build system uses Maturin to produce a mixed Python/Rust package; the Rust extension is exposed as `fastly_compute._fastly_compute_py`. _(evidenced by `[tool.maturin]` in `pyproject.toml`)_
