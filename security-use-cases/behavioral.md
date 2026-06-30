---
repo: security-use-cases
spec_type: behavioral
commit: cff1ada0f4f4e518788c9d5559822d717ba4a9cc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 8915d179abe02db61658168191858fbb9c8132958392fe9ba644280f230dd52c
generated_at: 2026-06-30T14:53:16.530250082+02:00
generator: specsync
---

## API Contracts

This repository is a collection of reference architectures, integration examples, and deployment templates for the Fastly Next-Gen WAF (NGWAF / Signal Sciences). It does not implement or expose a microservice API of its own.

No synchronous API endpoints are defined, served, or proxied by code within this repository. The Rust crates (`ngwaf-compute-integration`, `ngwaf-compute-interface`) are Fastly Compute edge programs that act as middleware/inspection delegates, not as API servers with documented routes.

_Not determinable from code._

---

## Event Schemas

No message broker integrations (Kafka, RabbitMQ, NATS, etc.) are present. No event producers or consumers are defined in any source file.

The Envoy deployment manifest configures a gRPC access-log stream directed at the `sigsci-agent-grpc` cluster (Signal Sciences agent on `127.0.0.1:9999`), but this is an infrastructure-level telemetry channel internal to the NGWAF agent — no application-level event schema is defined in this repository.

_Not determinable from code._

---

## Input / Output Formats

### Fastly Compute edge programs

Both Rust crates depend on the `fastly` SDK:

| Crate | `fastly` version | Additional deps |
|---|---|---|
| `ngwaf-compute-integration` | `0.11.2` | none |
| `ngwaf-compute-interface` | `0.11.9` | `serde_json 1.0.143` |

- `ngwaf-compute-interface` uses `serde_json`, indicating JSON serialization is used internally (likely for WAF signal/response parsing at the edge).
- Request and response handling follows the Fastly Compute ABI; HTTP/1.1 and HTTP/2 framing are managed by the Fastly runtime.

### NGWAF Agent sidecar (infrastructure layer)

| Integration pattern | Protocol to agent | Agent listen address |
|---|---|---|
| Module + Agent (k8s-module-agent, nginx, apache) | Unix Domain Socket (UDS) | `/sigsci/tmp/sigsci.sock` |
| Reverse proxy (k8s-rev-proxy, apache-apisix) | HTTP on `0.0.0.0:8080` | upstream forwarded to backend |
| Envoy ext_authz | gRPC over HTTP/2 | `127.0.0.1:9999` |

Content types and serialization formats for agent communication are determined by the Signal Sciences agent protocol, not by this repository's source code.

### NOOP origin (protect-cached-content)

Always returns HTTP `200`. No body format is defined.

---

## Error Handling

No application-level error handling code is directly visible in the extracted source. The following behaviours are documented by the README and deployment configs:

| Scenario | Behaviour |
|---|---|
| NGWAF returns BLOCK signal | WAF/agent response is returned directly to the client; request does not reach the origin or cache |
| NGWAF returns CHALLENGE signal | Challenge response is returned directly to the client |
| NGWAF returns ALLOW (no action) on a cache HIT | VCL restart is triggered; content is subsequently served from cache |
| NGWAF returns ALLOW on a cache MISS/PASS | Request is forwarded to origin normally |
| Envoy ext_authz failure | `failure_mode_allow: false` — requests are **denied** (fail closed) if the NGWAF agent is unreachable |
| Agent graceful shutdown | `preStop` lifecycle hook sleeps 30 seconds to drain in-flight requests before pod termination |

Specific HTTP error status codes returned by the NGWAF agent on block/challenge are controlled by NGWAF site configuration, not by code in this repository, and are therefore _not determinable from code._

---

## Versioning

No API versioning strategy (URI prefix, `Accept` header, schema evolution) is implemented by this repository. The repository itself is unversioned from an API perspective; it is a set of infrastructure templates rather than a versioned service.

Dependency pinning observed:

| Artifact | Pinned version |
|---|---|
| `fastly` (compute-integration) | `0.11.2` |
| `fastly` (compute-interface) | `0.11.9` |
| `serde_json` | `1.0.143` |
| `envoyproxy/envoy` image | `v1.22-latest` (floating tag) |
| `signalsciences/sigsci-agent` image | `latest` (floating tag, not pinned) |
