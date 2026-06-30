---
repo: security-use-cases
spec_type: functional
commit: cff1ada0f4f4e518788c9d5559822d717ba4a9cc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 8915d179abe02db61658168191858fbb9c8132958392fe9ba644280f230dd52c
generated_at: 2026-06-30T14:53:16.530250082+02:00
generator: specsync
---

## Business Purpose

This repository is a collection of reference implementations and integration patterns for deploying Fastly's Next-Generation Web Application Firewall (NGWAF, formerly Signal Sciences) across multiple infrastructure targets. It exists to help engineering teams protect web traffic—whether served from Fastly's edge CDN or from on-premises/cloud-native infrastructure—by providing tested, ready-to-use configuration artifacts, Terraform starters, Kubernetes manifests, and Fastly Compute (Rust) integrations that wire the NGWAF inspection pipeline into various traffic paths.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Web Application Firewall (WAF) integration and deployment patterns.
- **Core domain entities / aggregates this repository is responsible for:**
  - NGWAF Agent deployment unit (sidecar container / standalone process)
  - NGWAF module configuration (per web-server type: Apache, NGINX, Envoy, APISIX reverse proxy)
  - Fastly Compute edge programs (Rust Wasm) that route requests through the NGWAF inspection pipeline
  - VCL customisation for cached-content inspection (protect-cached-content)
  - Terraform-managed Fastly edge service configurations (edge-terraform-starter, ngwaf-terraform-edge-deploy, etc.)
- **Relationships to neighbouring contexts:**
  - **Upstream:** Fastly CDN / edge compute platform (VCL and Compute@Edge runtime); origin services that serve actual application content.
  - **Downstream:** NGWAF cloud control plane (Signal Sciences / Fastly NGWAF SaaS) for policy evaluation, telemetry upload, and configuration pulls.
  - **Peer:** Kubernetes control plane (for on-prem deployments); Terraform state backends; CI/CD pipelines (GitHub Actions).

## Use Cases / User Stories

- **Protect cached content at the edge:** As a platform engineer, I want cached CDN responses to be inspected by the NGWAF before being served, so that malicious or policy-violating requests are blocked or challenged even when the response is a cache HIT (implemented in `protect-cached-content` via VCL `vcl_pass` redirect + NGWAF inspection + restart-on-allow flow).
- **Block or challenge requests at the CDN edge:** As a security engineer, I want HTTP requests that traverse the Fastly edge to be evaluated by the NGWAF and returned a block/challenge response when a threat is detected, so that malicious traffic never reaches the origin (evidenced by BLOCK/CHALLENGED branch in `protect-cached-content` VCL sequences and Compute edge programs in `ngwaf-compute-integration` / `ngwaf-compute-interface`).
- **Deploy NGWAF alongside a Kubernetes Go application (module+agent pattern):** As a platform engineer, I want to run the NGWAF agent as a sidecar next to my Go application in Kubernetes using a Unix domain socket, so that all inbound HTTP traffic is inspected with minimal application-code changes (`on-prem-ngwaf-integrations/k8s-module-agent`).
- **Deploy NGWAF alongside a Kubernetes NGINX instance (NGINX module pattern):** As a platform engineer, I want to load the Fastly NGWAF NGINX dynamic module into my NGINX container and pair it with an agent sidecar, so that NGINX serves as an inspected reverse proxy (`on-prem-ngwaf-integrations/k8s-nginx-module-agent`).
- **Deploy NGWAF alongside Apache in a container:** As a platform engineer, I want to run the Fastly NGWAF Apache module and agent inside a Docker container, so that Apache traffic is inspected before responses are returned (`on-prem-ngwaf-integrations/apache`).
- **Deploy NGWAF as an Envoy external authorisation filter:** As a platform engineer, I want Envoy to delegate authorisation decisions to the NGWAF agent via gRPC `ext_authz`, so that the NGWAF can block requests before they reach the upstream cluster (`on-prem-ngwaf-integrations/envoy-deployment.yaml`).
- **Deploy NGWAF as an APISIX reverse proxy integration:** As a platform engineer, I want the NGWAF agent running in reverse-proxy mode behind an APISIX ingress, so that all routed traffic is WAF-inspected before reaching application backends (`on-prem-ngwaf-integrations/k8s-apache_apisix-rev-proxy`).
- **Deploy NGWAF in standalone Kubernetes reverse-proxy mode:** As a platform engineer, I want to run the NGWAF agent as a standalone reverse proxy pod in front of a backend NGINX service, so that I can add WAF protection without modifying the backend container (`on-prem-ngwaf-integrations/k8s-rev-proxy`).
- **Provision Fastly edge services with NGWAF via Terraform:** As a platform engineer, I want ready-made Terraform configurations that create and wire Fastly edge services with NGWAF, so that security infrastructure is version-controlled and reproducible (`edge-terraform-starter`, `ngwaf-terraform-edge-deploy`, `ngwaf-terraform-edge-deployment-unified-ui`, `edge-terraform-router`).

## Business Rules

- **Cache HIT must be forced through WAF inspection:** When a request results in a CDN cache HIT, VCL must redirect the request to `vcl_pass` before allowing it to be served, ensuring the NGWAF inspects it. Serving directly from cache without WAF inspection is not permitted. (evidenced by README flow diagrams and `protect-cached-content` description)
- **BLOCK or CHALLENGED response terminates the cache HIT flow:** If the NGWAF returns a BLOCK or CHALLENGED signal for a cache HIT request, the WAF response is returned to the client immediately; the cached content is not served. (evidenced by README Mermaid sequences)
- **Allowed cache HIT requires a VCL restart:** If the NGWAF does not block or challenge a cache HIT request, VCL must perform a restart so the request re-enters the cache lookup and is served from cache. Direct passthrough to origin without restart is not used for the cache HIT path. (evidenced by README)
- **Cache MISS path is unchanged:** For cache MISS or PASS requests, no modification to the standard NGWAF or VCL inspection flow is required; the existing agent-module integration handles inspection transparently. (evidenced by README "Cache MISS" section)
- **NOOP origin is required for cache HIT inspection:** The protect-cached-content pattern requires a dedicated NOOP origin service configured as a separate Fastly service; it must always return HTTP 200. This service counts toward the account's request quota. (evidenced by README)
- **Agent credentials must be supplied via Kubernetes Secrets:** All Kubernetes deployments supply `SIGSCI_ACCESSKEYID` and `SIGSCI_SECRETACCESSKEY` to the NGWAF agent container exclusively via `secretKeyRef` references; plaintext credentials in manifests are not used. (evidenced by all `deployment.yaml` files)
- **Agent graceful shutdown must be honoured:** The Kubernetes module+agent deployment defines a `preStop` hook that sleeps 30 seconds before agent termination, ensuring in-flight requests are processed before the pod is killed. (evidenced by `k8s-module-agent/deployment.yaml`)
- **Envoy `ext_authz` failure mode is strict:** The Envoy integration sets `failure_mode_allow: false`, meaning that if the NGWAF agent is unreachable, requests are denied rather than allowed through. (evidenced by `on-prem-ngwaf-integrations/envoy-deployment.yaml`)
- **NGWAF agent container uses a read-only root filesystem in the module+agent Kubernetes pattern:** The `sigsci-agent` container in `k8s-module-agent/deployment.yaml` sets `readOnlyRootFilesystem: true`; writable state is isolated to the shared `sigsci-tmp` emptyDir volume. (evidenced by `k8s-module-agent/deployment.yaml`)
- **Shared Unix domain socket volume is required for module+agent and NGINX module patterns:** The application container and the NGWAF agent must mount the same `sigsci-tmp` emptyDir volume so that the agent's RPC socket is accessible to the module. (evidenced by `k8s-module-agent/deployment.yaml`, `k8s-nginx-module-agent/deployment.yaml`)
- **Shielding affects NGWAF inspection location (inferred):** When Fastly shielding is enabled, WAF inspection for a cache HIT occurs at whichever PoP (edge or shield) holds the cache HIT, meaning WAF policy must be consistent across all PoPs. (evidenced by README "Note on shielding" section)
- **Fastly Compute programs depend on `fastly` crate ≥ 0.11.x:** Both Rust Wasm packages pin the `fastly` SDK dependency at version 0.11.2 and 0.11.9 respectively; earlier SDK versions are not supported. (evidenced by `Cargo.toml` files)
