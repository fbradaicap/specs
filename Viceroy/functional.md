---
repo: Viceroy
spec_type: functional
commit: d7ccf104ed0b5c4399a97a9442279c97705d749a
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ed522038d502957e706c70229783a2ab1afd79135845f526027725ea31a6bed8
generated_at: 2026-06-30T14:52:45.787430157+02:00
generator: specsync
---

## Business Purpose

Viceroy is a local development and testing tool for **Fastly Compute**, Fastly's serverless WebAssembly (Wasm) edge-computing platform. It provides a local simulation runtime that executes Fastly Compute guest Wasm programs on a developer's machine, faithfully emulating the Fastly Compute host ABI (defined via `witx`/WIT files). Its primary purpose is to allow developers to test and iterate on Compute applications without deploying to the Fastly edge network.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Fastly Compute local simulation / developer tooling.
- **Core domain entities / aggregates:**
  - **Guest Wasm module** — the compiled Compute application under test, loaded and executed via Wasmtime.
  - **Host ABI / hostcall implementations** — the Fastly Compute platform APIs (HTTP requests/responses, backends, dictionaries, KV stores, secrets, etc.) simulated locally via `wiggle`-generated stubs and `witx`/WIT definitions in `wasm_abi/`.
  - **Viceroy configuration** — TOML-based configuration describing backends, dictionaries, and other simulated platform resources (`viceroy-lib`).
  - **Local HTTP server** — the synthetic origin/listener that feeds synthetic requests into the guest Wasm module.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Fastly Compute platform ABI specification (canonical `witx`/WIT definitions in `wasm_abi/compute-at-edge-abi/`); the `fastly` Rust SDK crates (`fastly`, `fastly-sys`, `fastly-shared`) consumed by test fixtures.
  - **Downstream:** Developer CI pipelines and local developer workflows that invoke the `viceroy` CLI binary.

## Use Cases / User Stories

- **As a Fastly Compute developer**, I want to run my compiled Wasm application locally against a simulated Fastly edge environment so that I can test my service logic without deploying to Fastly's production network.
  - Capability: `viceroy` CLI binary (`cli/src/main.rs`) accepts a `.wasm` file, instantiates it via Wasmtime, and serves synthetic HTTP requests to it.
- **As a Fastly Compute developer**, I want to configure simulated backends, dictionaries, KV stores, and other platform resources in a local TOML configuration file so that my application code behaves as it would on the real platform.
  - Capability: `viceroy-lib` reads a TOML manifest describing backends and other resources.
- **As a Fastly Compute developer**, I want to run automated integration tests against my Wasm application in CI so that regressions are caught before deployment.
  - Capability: `viceroy-lib` is published as a Rust library crate (docs.rs) so it can be embedded in test harnesses; pre-built binaries are available via `cargo binstall`.
- **As a Fastly Compute developer**, I want the local runtime to surface fatal errors and Wasm traps clearly so that I can diagnose runtime failures.
  - Capability: `test-fatalerror-config` feature flag and the `cli/tests/trap-test` integration test exercise fatal hostcall error handling.
- **As a platform engineer at Fastly**, I want the Compute ABI to be formally specified in `witx`/WIT so that both the real edge runtime and Viceroy stay aligned with the canonical interface definition.
  - Capability: `wasm_abi/compute-at-edge-abi/` contains canonical `witx` definitions; `wasm_abi/adapter/` builds a component-model adapter wasm.
- **As a developer using components**, I want Viceroy to support the WebAssembly Component Model so that component-based Compute applications can also be tested locally.
  - Capability: `viceroy-component-adapter` crate in `wasm_abi/adapter/` produces a cdylib component adapter; `wit-component` and `wasm-encoder` are runtime dependencies of `viceroy-lib`.

## Business Rules

- **Wasm-only guest programs:** The service exclusively executes WebAssembly modules/components targeting the Fastly Compute ABI; non-Wasm artifacts are not accepted as guest programs. _(inferred from Wasmtime usage and project purpose)_
- **ABI conformance:** Guest programs must conform to the canonical `witx`/WIT ABI definitions in `wasm_abi/`; deviations will result in instantiation or linking failures at runtime. _(inferred)_
- **Configuration required for non-trivial platform features:** Backends, dictionaries, and other simulated resources must be declared in the TOML configuration file; references to undeclared resources will fail at runtime. _(inferred from `viceroy-lib` config loading)_
- **Fatal hostcall errors terminate the instance:** When a host-side call encounters a `FatalError`, the Wasm instance is terminated immediately (not gracefully); this is a defined and tested invariant (`test-fatalerror-config` feature, `trap-test`).
- **TLS support:** Outbound backend connections support TLS using the system native certificate store (`rustls-native-certs`) and can also be configured with PEM-format certificates (`rustls-pemfile`). _(inferred from dependencies)_
- **Minimum Rust toolchain version:** The library requires Rust ≥ 1.95 (`rust-version = "1.95"` in `Cargo.toml`).
- **Changelog entry required for every PR:** CI enforces that `CHANGELOG.md` is updated on every pull request unless the `skip-changelog` label is applied. _(from `.github/workflows/changelog.yml`)_
- **Release binaries are symbol-stripped with fat LTO:** Production release artifacts are built under the `release-lto-stripped` profile (fat LTO, single codegen unit, symbols stripped) to minimise binary size and maximise performance. _(from `Cargo.toml` profile definitions)_
- **Component adapter builds must not use Rust standard library features incompatible with a no-std/no-main environment:** The adapter crate disables debug assertions and overflow checks in dev builds and disables incremental compilation to produce a valid, minimal wasm adapter. _(from `wasm_abi/adapter/Cargo.toml`)_
