---
repo: keycloak-bcrypt
spec_type: functional
commit: 9b2c39431b99ae6a19eeda033dd8acba5c772bb7
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 34b481fd1be09917de008c5a9821d6465282d96dc7af18c5b209ab170f965a82
generated_at: 2026-06-30T14:54:03.729924115+02:00
generator: specsync
---

## Business Purpose

This service is a Keycloak Security Provider Interface (SPI) extension that adds BCrypt as a supported password hashing algorithm within Keycloak. It enables organizations using Keycloak for identity and access management to store and verify user passwords hashed with BCrypt, facilitating migration from systems that already use BCrypt-hashed passwords or enforcing BCrypt as the hashing policy. It is distributed as a JAR dropped into the Keycloak `providers/` directory (Keycloak ≥ 17) or as a pre-built Docker image.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Credential management / password hashing within the Keycloak Identity Provider context.
- **Core domain entities / aggregates:**
  - `PasswordCredentialModel` — the hashed credential stored by Keycloak for a user.
  - `PasswordPolicy` — the Keycloak policy that governs which hashing algorithm and iteration count are applied.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Keycloak core (`keycloak-server-spi`, `keycloak-server-spi-private`) — this extension is consumed by Keycloak's credential management subsystem via the `PasswordHashProvider` / `PasswordHashProviderFactory` SPI contracts.
  - **Downstream:** None detected; this is a leaf extension with no downstream consumers of its own.

## Use Cases / User Stories

- **As a Keycloak administrator**, I want to configure BCrypt as the password hashing algorithm via `Authentication → Policies → Password Policy` so that new user credentials are stored as BCrypt hashes.
- **As a Keycloak administrator**, I want to create a new user and set credentials so that I can verify the BCrypt provider is installed and working correctly.
- **As a system migrating to Keycloak**, I want existing BCrypt-hashed passwords (both `$2a$` and `$2y$` variants) to be recognised and verified by Keycloak so that users do not need to reset their passwords during migration.
- **As a platform engineer**, I want to install the provider as a JAR into an existing Keycloak deployment so that I can add BCrypt support without replacing the base image.
- **As a platform engineer**, I want to run a pre-built Docker image (`gleroy/keycloak-bcrypt`) so that I can deploy Keycloak with BCrypt support without a separate build step.

## Business Rules

- The BCrypt provider is registered under the algorithm identifier `"bcrypt"`; this string must be used as the value when setting the hashing algorithm password policy in Keycloak.
- The default cost factor (number of iterations) is **10** when no explicit iteration count is specified by the password policy.
- If the password policy specifies `hashIterations == -1` (Keycloak's sentinel for "use default"), the provider substitutes `DEFAULT_ITERATIONS` (10). (inferred)
- A stored credential passes the policy check only if both its recorded iteration count **and** its algorithm identifier match the current policy values; a mismatch fails the check.
- BCrypt salt is embedded in the encoded hash string; no separate salt field is stored (`new byte[0]` is passed as the salt value in `PasswordCredentialModel`).
- Password verification first attempts BCrypt `$2y$` (VERSION_2Y_NO_NULL_TERMINATOR with long-password truncation); if that fails, it falls back to BCrypt `$2a$` (VERSION_2A) to support legacy hashes. (inferred — supports backward compatibility with older BCrypt variants)
- Long passwords are handled via truncation strategy (`LongPasswordStrategies.truncate`) against BCrypt version `VERSION_2Y_NO_NULL_TERMINATOR`; passwords are **not** pre-hashed before BCrypt hashing.
- The provider targets Java 17 and Keycloak's SPI; the exact compatible Keycloak version range is parameterised at build time via `dependency.keycloak.version` and is not hardcoded. (inferred)
