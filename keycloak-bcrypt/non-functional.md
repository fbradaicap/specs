---
repo: keycloak-bcrypt
spec_type: non_functional
commit: 9b2c39431b99ae6a19eeda033dd8acba5c772bb7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 34b481fd1be09917de008c5a9821d6465282d96dc7af18c5b209ab170f965a82
generated_at: 2026-06-30T14:54:03.729924115+02:00
generator: specsync
---

## Performance

- BCrypt hashing is performed with a default cost factor of **10** (`DEFAULT_ITERATIONS = 10`), configurable per-realm via Keycloak's password policy (`HashIterations`). BCrypt at cost 10 is intentionally CPU-intensive; hashing and verification latency will scale exponentially with cost increases.
- Long passwords are truncated to the BCrypt 72-byte limit using `LongPasswordStrategies.truncate` (BCrypt 2Y variant), avoiding unbounded computation on very long inputs.
- No HTTP server, connection pools, caches, or tuning parameters are configured within this component — all such concerns are inherited from the Keycloak runtime into which this provider is installed.
- No explicit timeout configuration is present in the code or manifests for this component.

_Not determinable from code_ (throughput targets, latency SLOs, thread pool sizing).

## Scalability

- This component is a **Keycloak SPI plugin** (a JAR dropped into `providers/`); it carries no independent process, state, or replica configuration of its own. Scaling is entirely determined by the Keycloak cluster configuration.
- The provider implementation is **stateless** — `BCryptPasswordHashProvider` holds only two immutable primitive fields (`providerId`, `defaultIterations`) and creates no shared mutable state, making it safe for concurrent use within a scaled Keycloak deployment.
- Horizontal scaling, autoscaling, and partitioning are _Not determinable from code_ — they depend on the enclosing Keycloak deployment configuration, which is outside the scope of this component.

## Security

- **Password hashing algorithm**: BCrypt (at.favre.lib:bcrypt `0.10.2`) with Blowfish variant `2Y` (no null terminator). Falls back to verifying against `2A` variant to support legacy hashes.
- **Cost factor**: Default BCrypt cost of **10**, overridable via Keycloak realm password policy; higher values increase brute-force resistance.
- **Long-password handling**: Passwords are explicitly truncated at 72 bytes (`LongPasswordStrategies.truncate`) before hashing to prevent potential algorithmic abuse with extremely long inputs.
- **Salt storage**: BCrypt salt is embedded in the hash output; no separate salt storage is required or performed.
- **Transport security**: The `docker-compose.yml` exposes port `8443` (HTTPS) and mounts a keystore at `/opt/keycloak/conf/server.keystore`, indicating TLS is configured for the Keycloak server. Keycloak is started with `start` (production mode).
- **Secrets handling**: Admin credentials (`KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD`) and database credentials (`KC_DB_USERNAME`, `KC_DB_PASSWORD`) are passed via environment variables in the compose file. The compose example uses hardcoded placeholder values (`admin`/`admin`, `keycloak`/`keycloak`) — these are **not suitable for production**.
- **AuthN/AuthZ**: Entirely delegated to the Keycloak runtime. This plugin does not implement any authentication or authorisation logic beyond password hash encoding and verification.
- **Input validation**: No explicit input validation is performed in the plugin; null/empty password handling is delegated to the BCrypt library and Keycloak's credential framework.
- **SELinux label disabling**: `security_opt: label:disable` is present in `docker-compose.yml`, which relaxes container labelling — relevant only to the local development/test environment.

## Observability

- **Logging**: Uses `org.jboss.logging` (version `3.4.1.Final`) as the logging facade, consistent with Keycloak's internal logging infrastructure. No custom log statements are visible in the provided source samples; logging behaviour depends on Keycloak's log configuration.
- **Metrics, tracing, health/readiness endpoints**: _Not determinable from code._ This plugin introduces no independent metrics, distributed tracing instrumentation, or health endpoints. All such observability is provided by the enclosing Keycloak runtime.

## Reliability

- **Fallback verification**: The `verify()` method implements a two-pass verification strategy — it first attempts BCrypt `2Y` (no null terminator) and, on failure, retries with the `2A` variant. This provides backward compatibility with hashes created under older BCrypt configurations and avoids false authentication failures during hash algorithm migrations.
- **Idempotency**: Password encoding is deterministic given the same cost factor (BCrypt generates a random salt per call), and verification is a pure read operation — both are inherently safe to retry.
- **Failure handling**: No explicit exception handling, retry logic, or circuit-breaker patterns are implemented within the plugin. Error propagation is delegated to the Keycloak SPI framework.
- **Availability and recovery**: _Not determinable from code_ — the plugin has no independent process lifecycle; availability is governed entirely by the Keycloak deployment hosting it.
- **Statelessness**: The plugin holds no mutable state, eliminating state-related failure modes.
