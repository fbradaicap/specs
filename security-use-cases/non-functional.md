---
repo: security-use-cases
spec_type: non_functional
commit: cff1ada0f4f4e518788c9d5559822d717ba4a9cc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 8915d179abe02db61658168191858fbb9c8132958392fe9ba644280f230dd52c
generated_at: 2026-06-30T14:53:16.530250082+02:00
generator: specsync
---

## Performance

- **Envoy → NGWAF agent (gRPC/ext_authz) timeout:** 0.2 s, as set in the `envoy.access_loggers.grpc` `timeout` field of `envoy-deployment.yaml`.
- **Envoy → upstream edge cluster connect timeout:** 0.25 s (`connect_timeout: 0.25s` for `edge_cluster`).
- **Envoy → sigsci-agent-grpc connect timeout:** 0.25 s (`connect_timeout: 0.25s` for `sigsci-agent-grpc`).
- **NGINX worker connections:** 1,024 (`worker_connections: 1024` in the NGINX ConfigMap); `worker_processes` set to `auto` (one per CPU core).
- **NGINX keepalive timeout:** 65 s.
- **Cache utilisation:** Fastly VCL is explicitly configured to route cache-HIT requests through the NGWAF before serving from cache, reducing origin load; cache-MISS traffic is unaffected. A dedicated NOOP origin (always-200) is used as the WAF inspection target for cache-HIT flows to avoid touching the real origin.
- **Compile-time optimisations (Rust/Wasm compute components):** `ngwaf-compute-interface` uses `lto = "fat"` and `codegen-units = 1` for maximum release-build optimisation; both compute packages set `debug = 1` in the release profile.
- Connection pool sizing, thread pool sizing, and explicit latency/throughput targets are _Not determinable from code._

## Scalability

- **Kubernetes replica counts:** All Kubernetes `Deployment` manifests (`k8-ngwaf-module-agent`, `k8-nginx-module-agent-deployment`, `sigsci-revproxy-deployment`, `nginx-backend-deployment`, `ngwaf-apache-apisix-deployment`, `envoy-waf-deployment`) are configured with `replicas: 1`. Comments in `k8s-module-agent/deployment.yaml` note "Adjust replica count as needed."
- **Horizontal scaling:** Deployments are containerised and Kubernetes-managed, making horizontal scaling straightforward by increasing replica counts. The sigsci-agent sidecar/reverse-proxy pattern is stateless per pod.
- **Autoscaling:** No HorizontalPodAutoscaler or VerticalPodAutoscaler manifests are present. _Not determinable from code._
- **Fastly edge scaling:** The VCL/CDN layer inherits Fastly's globally distributed PoP infrastructure. Shield PoP behaviour is documented (inspection occurs at the PoP where the cache HIT is found).
- **Statelessness:** The NGWAF agent uses a shared ephemeral `emptyDir` volume (`/sigsci/tmp`) within a pod for the Unix domain socket (UDS); this is per-pod and non-persistent, supporting stateless horizontal scaling.
- **Partitioning/sharding:** _Not determinable from code._

## Security

- **Web Application Firewall (WAF):** The Fastly Next-Gen WAF (Signal Sciences / NGWAF) is the primary security control across all integration patterns. It inspects HTTP requests and can block or challenge traffic before it reaches the application or cache.
- **Deployment patterns covered:** Apache module, NGINX module, Envoy ext_authz (gRPC), Kubernetes sidecar (UDS RPC), Kubernetes reverse proxy, and Fastly Compute (Wasm) integration.
- **AuthN for NGWAF agent:** Agent credentials (`SIGSCI_ACCESSKEYID`, `SIGSCI_SECRETACCESSKEY`) are injected via Kubernetes `Secret` objects and referenced through `secretKeyRef` in all Kubernetes deployments; secrets are never hardcoded in manifests.
- **Transport security (upstream):** The Envoy `edge_cluster` upstream uses TLS (`envoy.transport_sockets.tls`) with SNI set to `http.edgecompute.app` on port 443. Internal Envoy ↔ sigsci-agent communication uses plain HTTP/2 on localhost (127.0.0.1:9999) within the same pod network namespace.
- **Ext_authz failure mode:** `failure_mode_allow: false` in the Envoy ext_authz filter — if the NGWAF agent is unreachable, requests are **denied** rather than allowed through (fail-closed).
- **Container hardening:** The `sigsci-agent` container in `k8s-module-agent/deployment.yaml` sets `readOnlyRootFilesystem: true`.
- **CI secrets handling:** GitHub Actions workflows pass `NGWAF_STAGING_ACCESSKEYID` and `NGWAF_STAGING_SECRETACCESSKEY` as repository secrets via the `staging` environment; they are not logged.
- **Input validation:** Delegated entirely to the NGWAF agent/module; no additional application-layer input validation is visible in the code.
- **CORS:** Envoy is configured with a broad CORS policy (`allow_origin_string_match: prefix: "*"`), permitting all origins. This is a permissive setting that may warrant review in production.
- **APK signing:** The NGINX Alpine Dockerfile fetches the Signal Sciences APK public key and registers it before installing packages.
- **Apt signing:** The Apache Dockerfile uses a GPG-signed apt repository for NGWAF module installation.

## Observability

- **NGWAF agent debug logging:** All Kubernetes deployments enable verbose agent logging via environment variables:
  - `SIGSCI_DEBUG_LOG_BLOCKED_REQUESTS=1`
  - `SIGSCI_DEBUG_LOG_WEB_INPUTS=1`
  - `SIGSCI_DEBUG_LOG_CONFIG_UPDATES=1`
  - `SIGSCI_DEBUG_LOG_CONFIG_UPLOADS=1`
  - `SIGSCI_DEBUG_LOG_WEB_OUTPUTS=1`
- **Envoy access logging:** Configured with two sinks — stdout (`envoy.access_loggers.stdout`) and a gRPC access log stream to the sigsci-agent (`envoy.http_grpc_access_log`, log name `sigsci-agent-grpc`). Logged request headers include `x-sigsci-request-id`, `x-sigsci-waf-response`, `accept`, `content-type`, `content-length`; response headers include `date`, `server`, `content-type`, `content-length`.
- **Envoy proxy logging level:** Set to `debug` in the deployment args (with a note to use `trace` for higher verbosity).
- **NGINX access/error logging:** Standard NGINX access log in `main` format to `/var/log/nginx/access.log`; error log to `/var/log/nginx/error.log` at `notice` level.
- **CI health check:** The `ngwaf-k8s-module-agent.yaml` workflow runs `kubectl rollout status`, `kubectl get pods`, `kubectl describe pods`, and `kubectl logs` after deployment, and waits 15 s for agent log upload.
- **Health/readiness endpoints:** _Not determinable from code._ (No dedicated `/healthz` or `/readyz` endpoint definitions found in manifests or source.)
- **Metrics and distributed tracing:** _Not determinable from code._

## Reliability

- **Fail-closed WAF (Envoy):** `failure_mode_allow: false` ensures that if the NGWAF ext_authz sidecar is unavailable, requests are blocked rather than bypassed, preserving security at the cost of availability.
- **Graceful shutdown:** The `sigsci-agent` container in `k8s-module-agent/deployment.yaml` defines a `preStop` lifecycle hook (`sleep 30`) to allow in-flight WAF inspections to complete before pod termination.
- **Cache-HIT idempotency:** The VCL restart mechanism after a WAF-allowed cache HIT is designed to be idempotent — the restarted request should hit the cache again without re-triggering a pass to the origin.
- **NOOP origin resilience:** A dedicated always-200 NOOP origin is required for WAF inspection of cached content; its unavailability would affect the cache-HIT WAF inspection path.
- **Kubernetes rollout timeout:** The CI workflow enforces a 30 s rollout timeout (`--timeout=30s`) for the staging deployment.
- **Retry and circuit-breaker patterns:** _Not determinable from code._
- **Persistence and recovery:** All agent state is ephemeral (`emptyDir` volumes); no persistent volumes are used. Pod restarts result in a clean agent state, relying on NGWAF cloud configuration being re-fetched on startup.
- **Replica redundancy:** All deployments use a single replica (`replicas: 1`); no multi-replica redundancy is configured in the provided manifests.
