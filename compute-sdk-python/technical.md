---
repo: compute-sdk-python
spec_type: technical
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:10.797313958+02:00
generator: specsync
---

## Tech Stack

- **Python**: ≥ 3.12 (target: 3.14 for type-checking); primary application language
- **Rust**: 2021 edition (crates `fastly-compute-py`, `wasiless`); used for native extension and WASM tooling
- **Build backend**: [Maturin](https://github.com/PyO3/maturin) ≥ 1.0, < 2.0 — builds the mixed Python/Rust package and emits a stable-ABI (`abi3`) wheel targeting CPython 3.12+
- **PyO3**: 0.28.3 with `abi3-py312` feature — Rust/Python FFI bridge
- **WSGI integration**: `fastly_compute.wsgi.WsgiHttpIncoming` — wraps WSGI-compatible frameworks (Flask, Bottle) for Fastly Compute
- **WebAssembly toolchain**: `componentize-py` (v0.23.0, bytecodealliance), `wit-component` 0.244, `wit-parser` 0.244, `wac-graph`/`wac-types`/`wac-parser` 0.8 — compiles Python apps to Wasm components
- **WIT bindings**: `wit-bindgen` 0.46.0 (Rust), `wit_world` Python bindings (generated); WIT interface definitions in `wit/`
- **Test framework**: pytest ≥ 9.0.3, with `syrupy` 5.0.0 (snapshot testing), `requests` ≥ 2.32.5
- **Linting/type-checking**: Ruff ≥ 0.12.11 (lint + format), Pyrefly ≥ 0.49.0
- **Notable Python runtime dependencies** (examples/test extras): Flask ≥ 3.1.2, Bottle ≥ 0.12.25

## Architecture Patterns

- **SDK / library architecture**: The repository is not a running microservice but a Python SDK for Fastly's Compute@Edge platform, packaged as a distributable wheel.
- **Mixed-language native extension**: The core compilation tooling is implemented in Rust (`crates/fastly-compute-py`) and exposed to Python via a PyO3 `cdylib` extension module (`fastly_compute._fastly_compute_py`). A companion CLI binary (`fastly_compute_py_build`) is also produced from the same crate.
- **WSGI adapter layer**: `fastly_compute.wsgi.WsgiHttpIncoming` bridges standard Python WSGI apps (Flask, Bottle) to the Fastly Compute HTTP incoming request model, allowing unmodified WSGI applications to run on the Compute platform.
- **WIT-driven host bindings**: Host APIs (HTTP, config stores, rate counters, penalty boxes, log endpoints, `compute_runtime`) are exposed through WIT interface definitions (`wit/`) compiled to Python bindings (`wit_world.imports.*`). The `FastlyResource[T]` generic base class (`fastly_compute/_resource.py`) provides uniform context-manager lifecycle management over these WIT-generated resource handles.
- **Error remapping decorator pattern**: `fastly_compute.runtime_patching.decorators.remap_wit_errors` converts WIT `result`-style errors (surfaced as `componentize_py_types.Err`) into typed Python exceptions deriving from `FastlyError`, providing a Pythonic error handling interface over the WIT ABI.
- **Test harness with Viceroy integration**: `fastly_compute.testing.ViceroyTestBase` / `AutoViceroyTestBase` manage a local [Viceroy](https://github.com/fastly/Viceroy) process (Fastly's local Compute runtime) with dynamic port allocation, and expose `get`/`post`/`request` helpers. A pytest plugin (`fastly_compute.pytest_plugin`) hooks into test failure events to surface Viceroy logs automatically.
- **Workspace layout**: Cargo workspace with two crates (`fastly-compute-py`, `wasiless`); Python package under `fastly_compute/`; examples are independent `pyproject.toml` projects that depend on the SDK via an editable path reference.

## Database & Data Ownership

This service owns no datastore or persistent schema. No database tables or models are defined. The SDK provides access to Fastly Compute host resources (config stores, rate counters, penalty boxes) that are owned and managed by the Fastly platform, not by this SDK itself.

## Dependencies

### Runtime (Python package, no mandatory install-time dependencies)
- **`fastly_compute` package**: declares zero mandatory Python dependencies (`dependencies = []` in `pyproject.toml`); all host integration is via the native extension and WIT-generated bindings.

### Runtime (native extension, Rust — compiled into the wheel)
| Crate | Version | Purpose |
|---|---|---|
| `componentize-py` | git tag v0.23.0 | Converts Python apps + WIT worlds to Wasm components |
| `wac-graph` / `wac-types` / `wac-parser` | 0.8 | Wasm component composition |
| `wit-parser` / `wit-component` | 0.244 (runtime), 0.219 (build) | WIT interface parsing and component encoding |
| `wasm-metadata` | 0.245 | Wasm binary metadata manipulation |
| `pyo3` | 0.28.3 (`abi3-py312`) | Python/Rust FFI |
| `clap` | 4 | CLI argument parsing (`fastly_compute_py_build` binary) |
| `anyhow`, `serde`/`serde_json`, `toml`, `log`/`env_logger`, `tempfile`, `futures`, `indexmap` | various | General Rust utilities |
| `wit-bindgen` | 0.46.0 (in `wasiless`) | WIT binding code generation for WASI stub crate |

### Build-time (Python)
| Package | Version | Purpose |
|---|---|---|
| `maturin` | ≥ 1.0, < 2.0 | Build backend; compiles Rust extension and packages wheel |
| `jinja2` | ≥ 3.1.6 | Used by `scripts/generate_patches` for code generation |

### Test/Dev extras (not distributed)
| Package | Version | Purpose |
|---|---|---|
| `pytest` | ≥ 9.0.3, < 10 | Test runner |
| `requests` | ≥ 2.32.5, < 3 | HTTP client for Viceroy-based integration tests |
| `syrupy` | 5.0.0 | Snapshot assertions |
| `tomli-w` | ≥ 1.0, < 2 | TOML serialisation in tests |
| `bottle` | ≥ 0.12.25 | WSGI framework used in test fixtures and examples |
| `flask` | ≥ 3.1.2, < 4 | WSGI framework used in examples and type-check CI |
| `ruff` | ≥ 0.12.11, < 0.13 | Linter and formatter |
| `pyrefly` | ≥ 0.49, < 0.50 | Static type checker |

### External runtime process
- **Viceroy** (Fastly's local Compute runtime): invoked as a subprocess by the test harness; not a Python package dependency but a required binary for running the test suite.

## Deployment Model

This repository is a **distributable SDK package**, not a deployed service. There is no Dockerfile, Kubernetes manifest, Helm chart, or Docker Compose file present. Deployment is via PyPI wheel distribution.

**Build**:
- `maturin build` (or `maturin develop` for editable installs) compiles the Rust extension (`crates/fastly-compute-py`) and packages everything into a `cp312-abi3-*` platform wheel.
- `make test` (per README) builds the WASM artifact and runs the pytest suite against a local Viceroy process.
- `uv run pytest -v -s` is the direct test invocation.

**Distribution**:
- Package name: `fastly-compute`, version `0.1.2`
- Exposes a console script entry point: `fastly-compute-py` → `fastly_compute.fastly_compute_py:main`
- Single wheel covers CPython ≥ 3.12 on a given platform (stable ABI).

**Ports / health endpoints**: _Not determinable from code._ (SDK library; no HTTP server is embedded in the package itself.)

**Environment configuration**: _Not determinable from code._ (No `.env`, k8s ConfigMap, or `os.environ` references found in the reviewed sources; runtime configuration of built Compute apps is platform-managed by Fastly.)
