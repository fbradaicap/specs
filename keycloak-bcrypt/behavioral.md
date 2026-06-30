---
repo: keycloak-bcrypt
spec_type: behavioral
commit: 9b2c39431b99ae6a19eeda033dd8acba5c772bb7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 34b481fd1be09917de008c5a9821d6465282d96dc7af18c5b209ab170f965a82
generated_at: 2026-06-30T14:54:03.729924115+02:00
generator: specsync
---

## API Contracts

This component is a **Keycloak SPI (Service Provider Interface) plugin**, not a standalone service with its own HTTP API. It is packaged as a JAR deployed into a Keycloak server and integrates via Keycloak's internal `PasswordHashProvider` SPI. No independent synchronous HTTP, gRPC, or GraphQL endpoints are exposed by this component.

The plugin registers itself under the provider ID `bcrypt` and is invoked internally by Keycloak's credential management subsystem. Interaction from operators is through the Keycloak Admin UI: `Authentication → Policies → Password Policy → add hashing algorithm = bcrypt`.

_No externally exposed API endpoints are determinable from code._

---

## Event Schemas

_Not determinable from code._

No asynchronous messaging, event topics, or queues are present. The component operates entirely as a synchronous in-process Keycloak SPI implementation.

---

## Input / Output Formats

The plugin operates via Keycloak's internal Java SPI contracts (`PasswordHashProvider`, `PasswordHashProviderFactory`). The data structures passed and returned are Keycloak's own model objects:

| Direction | Type | Description |
|-----------|------|-------------|
| Input | `String rawPassword` | Plaintext password string (char array internally) |
| Input | `int iterations` | BCrypt cost factor; if `-1`, defaults to `10` (`DEFAULT_ITERATIONS`) |
| Input | `PasswordPolicy` | Keycloak policy object carrying configured hash iterations |
| Input | `PasswordCredentialModel` | Keycloak credential model containing stored hash and algorithm metadata |
| Output | `PasswordCredentialModel` | Credential model created via `PasswordCredentialModel.createFromValues(providerId, new byte[0], iterations, encodedPassword)` |
| Output | `String` (encode) | BCrypt hash string in `$2y$` format |
| Output | `boolean` (verify / policyCheck) | Result of hash verification or policy compliance check |

**BCrypt specifics:**
- Algorithm variant: `VERSION_2Y_NO_NULL_TERMINATOR` (primary); fallback to `VERSION_2A` on verification failure.
- Long-password strategy: `LongPasswordStrategies.truncate(BCrypt.Version.VERSION_2Y_NO_NULL_TERMINATOR)`.
- Salt: embedded within the BCrypt hash string; the separate salt field in `PasswordCredentialModel` is always set to `new byte[0]`.
- Default cost factor: `10`; overridable via Keycloak password policy `hashIterations`.

No HTTP serialization format (JSON, Protobuf, Avro) is applicable.

---

## Error Handling

No HTTP error model is applicable. Error handling within the SPI implementation is minimal and not explicitly coded beyond what Keycloak's framework provides:

- If `policy.getHashIterations()` returns `-1`, the factory default of `10` is substituted silently.
- If `iterations` passed to `encode` or `encodedCredential` is `-1`, `defaultIterations` (`10`) is used.
- BCrypt verification first attempts `VERSION_2Y_NO_NULL_TERMINATOR`; on failure (non-match), it falls back to `VERSION_2A` before returning the final boolean result. No exception is thrown on a non-matching hash.
- No explicit validation exceptions, custom error payloads, or checked exception handling are present in the source.

_No HTTP status codes or structured error response bodies are determinable from code._

---

## Versioning

| Aspect | Detail |
|--------|--------|
| Plugin artifact version | `1.6.0` (defined in `build.gradle.kts`) |
| Keycloak version compatibility | Parameterised at build time via `dependency.keycloak.version` Gradle property; no version is hardcoded |
| Install path (Keycloak ≥ 17.0.0) | `${KEYCLOAK_HOME}/providers/` |
| Install path (Keycloak < 17.0.0) | `${KEYCLOAK_HOME}/standalone/deployments/` |
| SPI provider ID | `bcrypt` (constant; no versioning within the ID) |
| API versioning strategy | _Not applicable_ — no HTTP API is exposed. SPI contract versioning is governed by the Keycloak version in use. |
