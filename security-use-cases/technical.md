---
repo: security-use-cases
spec_type: technical
commit: cff1ada0f4f4e518788c9d5559822d717ba4a9cc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 8915d179abe02db61658168191858fbb9c8132958392fe9ba644280f230dd52c
generated_at: 2026-06-30T14:53:16.530250082+02:00
generator: specsync
---

## Tech Stack

| Component | Language / Runtime | Framework / Notable Libraries |
|---|---|---|
| `ngwaf-compute-integration` | Rust (edition 2021) | `fastly` crate v0.11.2 (Fastly Compute SDK) |
| `ngwaf-compute-interface` | Rust (edition 2021) | `fastly` crate v0.11.9, `serde_json` v1.0.143 |
| `on-prem-ngwaf-integrations / k8s-module-agent` | Go (latest toolchain) | `sigsci-module-golang` (Signal Sciences Go module, fetched via `go install`) |
| `on-prem-ngwaf-integrations / k8s-nginx-module-agent` | N/A (NGINX config) | `nginx:1.28.0-alpine` base, `nginx-module-fastly-nxs` dynamic module |
| `on-prem-ngwaf-integrations / apache` | N/A (shell/config) | `ubuntu:24.04` base, `sigsci-module-apache`, `sigsci-agent` (apt packages) |
| Kubernetes deployments (Envoy, rev-proxy, APISIX) | YAML manifests | `envoyproxy/envoy:v1.22-latest`, `signalsciences/sigsci-agent:latest`, `nginx:alpine` |

Terraform-based sub-directories (`edge-terraform-starter`, `ngwaf-terraform-edge-deploy`, `ngwaf-terraform-edge-deployment-unified-ui`, `edge-terraform-router`) are present but no manifests were surfaced for language/version analysis.

---

## Architecture Patterns

This repository is a **collection of reference implementation patterns and deployment examples** for integrating the Fastly Next-Gen WAF (NGWAF / Signal Sciences) across multiple deployment models. It is not a single deployable microservice. The following architectural patterns are demonstrated:

**1. Fastly Compute (Edge) Integration** (`ngwaf-compute-integration`, `ngwaf-compute-interface`)
- Rust WebAssembly programs compiled for the Fastly Compute platform.
- Implements an edge-side request interception layer that proxies inspection decisions to the NGWAF agent before serving or forwarding requests.

**2. Cached-Content WAF Inspection** (`protect-cached-content`)
- VCL-based pattern: on a cache HIT, the request is diverted via `vcl_pass` to the NGWAF for inspection before content is served from cache. A NOOP origin is required as a target for the WAF inspection request. On a cache MISS, standard NGWAF pass-through applies.

**3. Sidecar Agent Pattern** (Kubernetes deployments)
- `signalsciences/sigsci-agent` runs as a sidecar container sharing a Unix Domain Socket (UDS) volume (`/sigsci/tmp/sigsci.sock`) with the application container. The application module (Go, NGINX, Apache) communicates with the agent over the shared socket.

**4. Reverse Proxy Mode** (`k8s-rev-proxy`, `k8s-apache_apisix-rev-proxy`)
- `sigsci-agent` runs as a standalone reverse proxy (`SIGSCI_REVPROXY_LISTENER`) fronting an upstream backend service, performing WAF inspection on all inbound traffic.

**5. Envoy ext_authz Integration** (`envoy-deployment.yaml`)
- Envoy is configured with the `ext_authz` HTTP filter, delegating authorization decisions to the NGWAF agent via gRPC (cluster `sigsci-agent-grpc` on `127.0.0.1:9999`). gRPC access logging is also forwarded to the agent for full request telemetry.

**6. Infrastructure-as-Code** (`edge-terraform-*`, `ngwaf-terraform-*`)
- Terraform configurations for provisioning Fastly edge services and NGWAF deployments (contents not fully surfaced in the snapshot).

---

## Database & Data Ownership

This repository owns no datastores, schemas, or data models. No database migrations, ORM entity definitions, or persistent storage configurations are present in any sub-project.

---

## Dependencies

### Runtime Dependencies

| Dependency | Type | Used By | Notes |
|---|---|---|---|
| `fastly` crate v0.11.2 | Runtime (Rust) | `ngwaf-compute-integration` | Fastly Compute SDK for Rust; targets Wasm runtime |
| `fastly` crate v0.11.9 | Runtime (Rust) | `ngwaf-compute-interface` | Same SDK, newer patch version |
| `serde_json` v1.0.143 | Runtime (Rust) | `ngwaf-compute-interface` | JSON serialisation/deserialisation |
| `signalsciences/sigsci-agent` (Docker image, `latest`) | Runtime | All Kubernetes deployments | Fastly NGWAF agent; performs WAF inspection, telemetry upload |
| `sigsci-module-apache` (apt package) | Runtime | `on-prem-ngwaf-integrations/apache` | Apache HTTPD NGWAF integration module |
| `sigsci-agent` (apt package) | Runtime | `on-prem-ngwaf-integrations/apache` | NGWAF agent for on-prem Apache |
| `nginx-module-fastly-nxs` (apk package) | Runtime | `k8s-nginx-module-agent` | NGINX dynamic module for NGWAF |
| `sigsci-module-golang` (`examples/helloworld`) | Runtime | `k8s-module-agent` | Go NGWAF module example application |
| `envoyproxy/envoy:v1.22-latest` | Runtime | `envoy-deployment.yaml` | Envoy proxy with `ext_authz` filter for NGWAF gRPC integration |
| `nginx:alpine` / `nginx:1.28.0-alpine` | Runtime | Multiple K8s deployments | Upstream backend or NGINX module base |
| Fastly NGWAF Cloud API | External service | All deployments | Agent authenticates via `SIGSCI_ACCESSKEYID` / `SIGSCI_SECRETACCESSKEY`; uploads telemetry and retrieves rule configurations |
| `http.edgecompute.app` | External service | `envoy-deployment.yaml` | Fastly edge compute upstream (TLS, port 443) |

### Build / CI Dependencies

| Dependency | Type | Used By | Notes |
|---|---|---|---|
| `golang:latest` (Docker image) | Build | `k8s-module-agent/Dockerfile` | Go toolchain for building the helloworld example |
| `engineerd/setup-kind@v0.6.2` | CI (GitHub Actions) | `ngwaf-k8s-module-agent.yaml` | Spins up a local Kubernetes cluster for integration testing |
| `azure/setup-kubectl@v4` | CI (GitHub Actions) | `ngwaf-k8s-module-agent.yaml` | Provides `kubectl` in the CI runner |
| `actions/checkout@v3` | CI (GitHub Actions) | `ngwaf-k8s-module-agent.yaml` | Repository checkout step |

### Secrets / Environment Configuration

All Kubernetes deployments require `SIGSCI_ACCESSKEYID` and `SIGSCI_SECRETACCESSKEY` supplied via Kubernetes `Secret` objects. The GitHub Actions workflow reads these from repository environment secrets (`NGWAF_STAGING_ACCESSKEYID`, `NGWAF_STAGING_SECRETACCESSKEY`).

---

## Deployment Model

### Fastly Compute (Rust/Wasm) Sub-projects
- **Build**: `cargo build --release` (Wasm target implied by the `fastly` SDK); `publish = false` in both `Cargo.toml` files prevents accidental crate publication.
- **Release profile**: debug symbols retained (`debug = 1`); `ngwaf-compute-interface` additionally sets `codegen-units = 1` and `lto = "fat"` for maximum optimisation.
- **Deployment target**: Fastly Compute platform (Wasm binary uploaded via Fastly CLI or Terraform). No container or Kubernetes manifest is involved for these sub-projects.
- **Ports / health endpoints**: _Not determinable from code._

### Kubernetes / Container Sub-projects

| Sub-project | Image Built | Orchestration | Exposed Port | Health Endpoint |
|---|---|---|---|---|
| `k8s-module-agent` | `signalsciences/example-helloworld:latest` (from `golang:latest`) | `k8s-module-agent/deployment.yaml`; 1 replica; `kind` cluster in CI | Service port 80 → container port 8000 | _Not determinable from code._ |
| `k8s-nginx-module-agent` | `my-local-images/nginx-module:latest` (from `nginx:1.28.0-alpine`) | `k8s-nginx-module-agent/deployment.yaml`; 1 replica; `NodePort` | Service NodePort 80 → container port 80 | _Not determinable from code._ |
| `apache` | Built from `ubuntu:24.04` | No K8s manifest present; standalone Docker container | _Not determinable from code._ | _Not determinable from code._ |
| `k8s-rev-proxy` | `signalsciences/sigsci-agent:latest` + `nginx:alpine` | `k8s-rev-proxy/deployment.yaml`; 2 separate Deployments (agent, nginx); `NodePort` | sigsci-revproxy Service: `NodePort` 80 → container 8080 | _Not determinable from code._ |
| `k8s-apache_apisix-rev-proxy` | `signalsciences/sigsci-agent:latest` + `nginx:alpine` | `k8s-apache_apisix-rev-proxy/deployment.yaml`; APISIX `Ingress`; 1 replica | Service `ClusterIP` port 80 → agent port 8080 | _Not determinable from code._ |
| `envoy-deployment` | `envoyproxy/envoy:v1.22-latest` + `signalsciences/sigsci-agent:latest` | `envoy-deployment.yaml`; 1 replica; sidecar pattern | Envoy listens on container port 10000; agent gRPC on `127.0.0.1:9999` | _Not determinable from code._ |

### CI/CD
- GitHub Actions workflow (`.github/workflows/ngwaf-k8s-module-agent.yaml`) targets the `staging` environment, triggers on `workflow_dispatch`, builds and loads the helloworld Docker image into a `kind` cluster, deploys via `kubectl apply`, and validates with a `curl` smoke test to `http://localhost:10000/anything/k8stest` expecting HTTP 200.

### Environment Configuration (all K8s deployments)
- `SIGSCI_ACCESSKEYID` and `SIGSCI_SECRETACCESSKEY`: required; injected from Kubernetes `Secret` resources.
- `SIGSCI_DEBUG_LOG_*` flags: set to `"1"` across all example deployments (intended for demonstration/debug; should be disabled in production).
- `SIGSCI_REVPROXY_LISTENER`: configures the agent's reverse proxy listener address and upstream URL (used in rev-proxy deployments).
- Agent RPC socket path: `/sigsci/tmp/sigsci.sock` (shared `emptyDir` volume between app and agent containers).
