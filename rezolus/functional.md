---
repo: rezolus
spec_type: functional
commit: f8beb6fce7394ff5421f0ad9371adf2bd25903d7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 46fbc1f0c5073dc355a6ecb735431d5c6d3a5672528bcd134a89f460a7891e0e
generated_at: 2026-06-30T14:54:33.524586697+02:00
generator: specsync
---

## Business Purpose

Rezolus is a high-resolution Linux systems performance telemetry agent built by Fastly. It collects detailed, low-overhead performance metrics across CPU, scheduler, block I/O, network, system calls, and containers using eBPF instrumentation. It exists to serve performance engineering workflows (lab/production workload characterisation) and DevOps/SRE troubleshooting scenarios where fine-grained, sub-second visibility into system behaviour is required.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Systems observability and performance telemetry collection on Linux hosts.
- **Core domain entities / aggregates:**
  - **Metrics** — time-series counters, gauges, and histograms collected per sampler domain (CPU, scheduler, block I/O, network, syscalls, containers, GPU via NVML).
  - **Artifact** — a Parquet file produced by the Recorder or Hindsight mode, representing a captured metrics window.
  - **Ring buffer (Hindsight)** — a rolling, high-resolution on-disk buffer of recent metrics.
  - **Sampler** — a pluggable unit responsible for collecting metrics within one domain (evident from eBPF probe structure and `linkme`-based registration).
- **Neighbouring contexts:**
  - **Downstream — Prometheus / external observability stacks:** The Exporter mode exposes a Prometheus-compatible scrape endpoint, making Rezolus a data producer for external monitoring pipelines.
  - **Downstream — Incident investigation tooling / analysts:** Hindsight artifacts are consumed by operators performing post-incident root-cause analysis.
  - No upstream service dependencies are detected in the code.

## Use Cases / User Stories

- **As a performance engineer**, I want to run the Rezolus Agent to continuously collect high-resolution system metrics, so that I can characterise workload behaviour in lab or production environments.
- **As a performance engineer**, I want to use the Recorder to capture a timed Parquet snapshot (`rezolus record --interval 1s --duration 15m <agent-url> <file>`), so that I can analyse a specific test run offline.
- **As a performance engineer**, I want to open a Parquet artifact in the Viewer (`rezolus view <file>`), so that I can explore metrics interactively in a local web-based dashboard.
- **As a DevOps/SRE engineer**, I want the Exporter to expose metrics on a Prometheus-compatible endpoint with optional histogram-to-summary conversion, so that I can integrate Rezolus data into my existing observability stack without prohibitive storage costs.
- **As an SRE responding to a production incident**, I want the Hindsight mode to maintain a rolling ring buffer on disk, so that I can snapshot high-resolution metrics retroactively after an unexpected performance event has already occurred.
- **As a system operator**, I want Rezolus to run as a managed systemd service (agent, exporter, hindsight variants), so that it is automatically started and supervised on the host.
- **As a container platform operator**, I want Rezolus to collect container-level performance metrics via eBPF, so that I can attribute resource usage and latency to individual containers.

## Business Rules

- **Linux-only instrumentation:** eBPF-dependent samplers (`libbpf-rs`, `perf-event2`, `nvml-wrapper`) are compiled only on `target_os = "linux"`; the service requires Linux kernel 5.8 or later.
- **Supported architectures:** Deployable only on x86_64 and ARM64 hosts (per README; enforcement mechanism not visible in code).
- **Parquet as artifact format:** Recorder and Hindsight outputs are always written as Apache Parquet files; the Viewer consumes only Parquet artifacts.
- **Histogram summarisation in Exporter:** The Exporter may convert histogram distributions to summary percentiles; this is a user-configured option intended to reduce downstream storage requirements (inferred from README and `metriken-exposition` dependency).
- **Hindsight ring-buffer semantics:** The buffer is rolling (oldest data is evicted); a snapshot must be explicitly triggered to persist data — data is not automatically saved on incident detection (inferred).
- **Configuration via TOML:** Each operating mode (agent, exporter, hindsight) has a distinct TOML configuration file deployed to `/etc/rezolus/`; misconfiguration cannot be compensated at runtime (inferred from file layout).
- **Single binary, multiple sub-commands:** All modes (`agent`, `record`, `view`, exporter, hindsight) are dispatched from a single `rezolus` binary via CLI sub-commands (evident from `clap` with `derive` feature and `[[bin]]` declaration).
- **Release build optimisations:** Production and benchmark profiles enforce LTO and single codegen unit, ensuring minimal runtime overhead consistent with the low-overhead telemetry promise.
- **Dual licensing:** All source is MIT OR Apache-2.0 unless otherwise noted; redistribution must comply with both licence options.
