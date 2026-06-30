---
repo: Viceroy
spec_type: technical
commit: d7ccf104ed0b5c4399a97a9442279c97705d749a
model: claude-sonnet-4-6
prompt_version: v1
input_hash: ed522038d502957e706c70229783a2ab1afd79135845f526027725ea31a6bed8
generated_at: 2026-06-30T14:52:45.787430157+02:00
generator: specsync
---

## Tech Stack

- **Language:** Rust (minimum toolchain version `1.95` for `viceroy-lib`; edition `2024`)
- **Runtime/Execution Engine:** [Wasmtime](https://wasmtime.dev/) `39.0.2` (with `call-hook` feature) — used to execute WebAssembly guest programs
- **WebAssembly tooling:** `wat` `1.250.0`, `wasmparser` `0.250.0`, `wasm-encoder` `0.250.0`, `wit-component` `0.250.0`, `wiggle` `39.0.2`, `walrus` `0.23.3`, `wasm-metadata` `0.250.0`
- **WASI support:** `wasmtime-wasi` `39.0.2`, `wasmtime-wasi-io` `39.0.2`, `wasmtime-wasi-nn` `39.0.2`
- **HTTP / networking:** Hyper `0.14.26` (full features), `http` `0.2.8`, `http-body` `0.4.5`
- **Async runtime:** Tokio `1.49.0` (full features)
- **TLS:** Rustls `0.21`, `tokio-rustls` `0.24.1`, `rustls-native-certs`, `rustls-pemfile`
- **Serialisation:** `serde` `1.0`, `serde_json` `1.0`, `toml` `0.5`
- **CLI parsing:** Clap `4.x` (derive feature)
- **Observability/tracing:** `tracing` `0.1`, `tracing-futures` `0.2`, `tracing-subscriber` `0.3`
- **Caching:** `moka` `0.12` (async, in-process)
- **Compression:** `flate2` `1.0`
- **Other notable libraries:** `anyhow`, `thiserror`, `bytes`, `futures`, `regex`, `base64`, `url`, `lazy_static`, `pin-project`, `async-trait`, `cranelift-entity`
- **Component adapter sub-crate:** `viceroy-component-adapter` (cdylib, compiled with `wit-bindgen-rust-macro`, `wasi`, `bitflags`; targets Wasm component model)
- **Test fixtures:** compiled as separate Wasm guest programs using the `fastly` SDK `0.11.11`
- **Build/CI base image:** Ubuntu Bionic with stable Rust (via `rustup`)

## Architecture Patterns

**Local Emulation / Developer-Tooling Daemon** — Viceroy is a local testing server that emulates the Fastly Compute platform. It is not a networked microservice in the traditional sense; it is a developer tool that loads a guest WebAssembly program, provides Fastly host-call ABI stubs, and serves HTTP requests to that guest.

Key internal architectural elements:

- **Library + CLI split:** Core logic lives in the `viceroy-lib` crate (`src/`); the `viceroy` binary (`cli/`) is a thin CLI wrapper over the library. This allows embedding the library in tests (e.g. `trap-test`).
- **Host ↔ Guest ABI layer:** Host-call bindings are generated via `wiggle` against `witx` ABI definitions in `wasm_abi/compute-at-edge-abi/`. This layer translates Wasmtime host-function calls into Rust implementations that simulate Fastly edge behaviour.
- **Wasm Component Model adapter:** `wasm_abi/adapter/` contains a cdylib adapter (`viceroy-component-adapter`) that bridges the Wasm component model to the Compute ABI; it is compiled to `wasm32` separately.
- **Async, event-driven request handling:** Tokio async runtime powers both the inbound HTTP listener (Hyper server) and outbound backend calls, with futures-based concurrency.
- **In-process caching:** `moka` provides async in-memory caching, simulating Fastly's cache API.
- **TLS termination and backend simulation:** Rustls + `tokio-rustls` enable TLS for simulated backend connections.
- **Configuration-driven:** Service topology (backends, dictionaries, geolocation stubs, etc.) is driven by TOML configuration files parsed at startup.
- **Integration test fixtures:** Wasm guest programs in `test-fixtures/` are compiled separately (via Makefile) and used as black-box integration test subjects.

## Database & Data Ownership

This service owns **no persistent datastore**. There are no database migrations, ORM models, or external database dependencies. All state is ephemeral and in-process (e.g., the `moka` async cache holds simulated edge-cache entries for the lifetime of the process).

## Dependencies

### Runtime dependencies

| Dependency | Kind | Purpose |
|---|---|---|
| Wasmtime `39.0.2` + wasi/wasi-io/wasi-nn | Runtime | Execute guest Wasm programs; provide WASI interfaces |
| Hyper `0.14.26` | Runtime | HTTP/1.1 server and client for inbound request handling and backend calls |
| Tokio `1.49.0` | Runtime | Async task scheduling and I/O |
| Rustls `0.21` / `tokio-rustls` | Runtime | TLS for simulated backend connections |
| `moka` `0.12` | Runtime | In-process async cache (simulates Fastly edge cache) |
| `wiggle` `39.0.2` | Runtime | Host-call glue generated from witx ABI definitions |
| `walrus` `0.23.3` | Runtime | Wasm binary manipulation (module rewriting) |
| `wasm-encoder`, `wasmparser`, `wit-component` | Runtime | Wasm/component model parsing and emission |
| `fastly-shared` `0.11.5` | Runtime | Shared type definitions matching the Fastly Compute ABI |
| `tracing` / `tracing-subscriber` | Runtime | Structured logging and diagnostics |
| `clap` `4.x` | Runtime (CLI) | Command-line argument parsing |
| `serde` / `serde_json` / `toml` | Runtime | Configuration file and data serialisation |
| `flate2` | Runtime | Gzip/deflate compression simulation |
| `hyper` + `tls-listener` | Dev/test | Integration test HTTP client and TLS listener |

### Build / dev dependencies

| Dependency | Kind | Purpose |
|---|---|---|
| `proptest` `1.6`, `proptest-derive` | Dev | Property-based tests in `viceroy-lib` |
| `tempfile` `3.x` | Dev | Temporary file management in tests |
| `fastly` / `fastly-sys` `0.11.11` | Dev (test fixtures) | Guest-side SDK used to compile test Wasm programs |

### External service dependencies

_Not determinable from code._ — Viceroy simulates Fastly backends locally; actual external service calls depend on the TOML configuration provided at runtime by the developer, not on hard-coded service dependencies within Viceroy itself.

## Deployment Model

**Viceroy is a local developer CLI tool, not a deployed service.** It is distributed as a pre-built binary rather than a running infrastructure component.

- **Binary name:** `viceroy` (entry point: `cli/src/main.rs`)
- **Distribution:** Pre-built release binaries hosted on GitHub Releases, installable via `cargo binstall`. Release assets are named `viceroy_v{version}_{platform}.tar.gz` and target:
  - `x86_64-apple-darwin` (darwin-amd64)
  - `aarch64-apple-darwin` (darwin-arm64)
  - `x86_64-unknown-linux-gnu` (linux-amd64)
  - `aarch64-unknown-linux-gnu` (linux-arm64)
  - `x86_64-pc-windows-msvc` (windows-amd64)
- **Release build profile:** `release-lto-stripped` — fat LTO, single codegen unit, symbols stripped, optimised for minimal binary size.
- **Container image (CI/build only):** `ubuntu:bionic`-based Dockerfile installs build tooling (gcc, curl, git, libssl-dev) and stable Rust via `rustup`. This image is used for building, not for running Viceroy in production.
- **Ports:** _Not determinable from code._ — The listening port is configurable at runtime via CLI flags.
- **Orchestration (Kubernetes/Helm/Compose):** None — not applicable for a local CLI tool.
- **Health/readiness endpoints:** _Not determinable from code._
- **Environment configuration:** Primarily via CLI arguments (Clap) and a TOML configuration file specifying backends, dictionaries, and other Fastly platform stubs. `LD_LIBRARY_PATH=/usr/local/lib` is set in the build image for consistent Cargo caching.
