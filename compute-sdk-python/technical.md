---
repo: compute-sdk-python
spec_type: technical
commit: 10d556298a332e54635734509db5b7e60e4e181d
model: claude-sonnet-4-6
prompt_version: v1
input_hash: e07c5edd4a6503c59acf412f2b47c47848f718785c9d441753ea4a4f4e9e1ef6
generated_at: 2026-06-30T14:57:33.154103392+02:00
generator: specsync
---

## Tech Stack

- **Primary Language**: Python ≥ 3.12 (targeting up to 3.14 for type checking); Rust (edition 2021/2024)
- **Build System**: [Maturin](https://www.maturin.rs/) ≥ 1.0, < 2.0 — builds a mixed Python/Rust extension wheel using the `abi3-py312` stable ABI, emitting a single `cp312-abi3-*` wheel usable on CPython 3.12+
- **Rust Crates**:
  - `fastly-compute-py` (v0.1.2) — `cdylib`/`rlib` exposing a Python extension module via [PyO3](https://pyo3.rs/) 0.28.3 and a `fastly_compute_py_build` CLI binary
  - `wasiless` (v0.1.0) — `cdylib` providing minimal/trapping WASI interface implementations via `wit-bindgen` 0.46
- **Notable Rust Dependencies**: `componentize-py` (git, tag v0.23.0 from bytecodealliance), `wac-graph/wac-types/wac-parser` 0.8, `wit-parser`/`wit-component` 0.244, `wasm-metadata` 0.245, `pyo3` 0.28.3, `clap` 4, `serde`/`serde_json`, `toml`, `tempfile`, `anyhow`, `indexmap`
- **Python Runtime Dependencies** (core `fastly-compute` package): none at runtime
- **Python Test Dependencies**: `pytest` ≥ 9.0.3, `requests` ≥ 2.32.5, `bottle` ≥ 0.12.25, `syrupy` 5.0.0, `tomli-w` ≥ 1.0
- **Python Dev Dependencies**: `ruff` ≥ 0.12.11, `pyrefly` ≥ 0.49, `jinja2` ≥ 3.1.6, `maturin` ≥ 1.11.5
- **Example Application Frameworks**: [Bottle](https://bottlepy.org/) ≥ 0.12.25, [Flask](https://flask.palletsprojects.com/) ≥ 3.1.2

---

## Architecture Patterns

- **SDK / Library Package** — `compute-sdk-python` is not a running service; it is a Python SDK distributed as a wheel that application authors import to build and deploy Fastly Compute (WebAssembly/WASM) edge functions.
- **Mixed Native Extension**: A Rust `cdylib` compiled by Maturin and exposed as `fastly_compute._fastly_compute_py`. Python consumer code calls into the Rust layer for build-time WASM composition and toolchain utilities.
- **WIT/Component Model Integration**: The SDK bridges Python WSGI applications to the [WebAssembly Component Model](https://component-model.bytecodealliance.org/) via `componentize-py`. WIT interface bindings (`wit_world.imports.*`) are generated and shipped so user code can call Fastly Compute host APIs (e.g., `compute_runtime`, `types`).
- **WSGI Bridge**: `fastly_compute.wsgi.WsgiHttpIncoming` wraps standard Python WSGI apps (Bottle, Flask) as Fastly Compute HTTP handlers, making the SDK framework-agnostic.
- **Resource Wrapper Pattern**: `FastlyResource[T]` (generic base class) provides uniform context-manager lifecycle management over WIT-generated resource handles (`ConfigStore`, `RateCounter`, `PenaltyBox`, `LogEndpoint`, etc.).
- **Error Remapping Layer**: `remap_wit_errors` decorator (`fastly_compute.runtime_patching.decorators`) translates WIT result-based errors (`componentize_py_types.Err`) into idiomatic Python exception hierarchies (`FastlyError`, `UnexpectedFastlyError`).
- **Test Infrastructure**: `ViceroyTestBase` / `AutoViceroyTestBase` spin up a local [Viceroy](https://github.com/fastly/Viceroy) (Fastly Compute local runtime) process against a built `.wasm` file, with dynamic port allocation. A pytest plugin (`fastly_compute.pytest_plugin`) captures and surfaces Viceroy logs on test failure.
- **Code Generation**: `scripts/generate_patches/` (Jinja2-driven) generates runtime patches (`fastly_compute/runtime_patching/patches.py`).

---

## Database & Data Ownership

This service owns no datastore and no persistent schema. No database tables, migrations, or data models are present. The SDK provides access to Fastly-hosted stores (e.g., `ConfigStore`) via WIT host API bindings, but those stores are owned and managed externally by the Fastly platform.

---

## Dependencies

### Runtime (published wheel — zero Python dependencies)
The core `fastly-compute` package declares **no runtime Python dependencies** (`dependencies = []` in `pyproject.toml`). All host API access occurs through the compiled Rust native extension and WIT bindings at the Fastly Compute edge.

### Build / Compilation
| Dependency | Role |
|---|---|
| `maturin` ≥ 1.0 | Python/Rust wheel builder |
| `componentize-py` v0.23.0 | Converts Python + WIT → WASM component |
| `wac-graph` / `wac-types` / `wac-parser` 0.8 | WebAssembly component composition |
| `wit-parser` / `wit-component` 0.244 | WIT interface parsing and component encoding |
| `wasm-metadata` 0.245 | WASM module metadata manipulation |
| `pyo3` 0.28.3 | Rust ↔ Python FFI |
| `wit-bindgen` 0.46 (`wasiless` crate) | WASI stub generation |
| `jinja2` ≥ 3.1.6 | Template engine for patch generation scripts |

### Test
| Dependency | Role |
|---|---|
| `pytest` ≥ 9.0.3 | Test runner |
| `requests` ≥ 2.32.5 | HTTP client used in `ViceroyTestBase` |
| `bottle` ≥ 0.12.25 | WSGI framework used in test fixtures |
| `syrupy` 5.0.0 | Snapshot testing |
| `tomli-w` ≥ 1.0 | TOML serialisation in test helpers |
| Viceroy (external binary) | Local Fastly Compute runtime; must be present on `PATH` |

### External Platform
- **Fastly Compute host APIs** — accessed entirely through WIT bindings at edge runtime; no network calls from the SDK itself at build time.

---

## Deployment Model

This repository produces a **distributable Python package** (wheel), not a deployed service.

**Build**:
```
maturin build --features pyo3/abi3-py312
```
Maturin compiles the `fastly-compute-py` Rust crate and packages it together with the `fastly_compute/` Python sources into a single `cp312-abi3-*` wheel (stable ABI, works on CPython 3.12+).

**Installed CLI entry point**:
```
fastly-compute-py  →  fastly_compute.fastly_compute_py:main
```
This CLI (backed by the `fastly_compute_py_build` Rust binary) drives the WASM component build pipeline.

**Container / Kubernetes / Ports**: _Not determinable from code._ No Dockerfile, Docker Compose file, Kubernetes manifests, or Helm charts are present. The SDK itself runs inside the Fastly Compute (WASM) edge environment, not in a container.

**Testing / local development**:
```bash
make test           # build WASM + run pytest
uv run pytest -v -s # run tests directly (requires pre-built WASM)
```
Tests require a compiled `.wasm` file (default: `app.wasm`) and a Viceroy binary available in the environment. Viceroy is launched as a subprocess on a dynamically allocated local port.

**Environment configuration**: _Not determinable from code._ No `.env`, `k8s` ConfigMap, or explicit `os.environ` reads are surfaced in the provided manifests beyond `REQUEST_METHOD` and `PATH_INFO` WSGI environ keys.
