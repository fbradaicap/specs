---
repo: boulder
spec_type: functional
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:57:58.750743927+02:00
generator: specsync
---

## Business Purpose

Boulder is the certificate authority (CA) software that powers Let's Encrypt, implementing the ACME (Automated Certificate Management Environment) protocol (RFC 8555) to automate the issuance, renewal, and revocation of TLS/SSL certificates. It provides the full lifecycle management of X.509 certificates—from account registration and domain control validation through certificate issuance, OCSP/CRL status publication, and revocation. The service exists to enable domain owners to obtain trusted TLS certificates automatically without manual CA interaction.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Public Key Infrastructure / Automated Certificate Management
- **Core domain entities / aggregates:**
  - `Registration` — ACME account (linked to a JWK public key and contact info)
  - `Authorization` (`authz2`) — domain control validation authorization, with associated `Challenge` objects
  - `Order` — certificate issuance request grouping one or more `Authorization`s
  - `Certificate` / `Precertificate` — issued X.509 certificates and their pre-issuance counterparts
  - `CertificateStatus` — revocation and OCSP state of a certificate
  - `CRLShard` — a time/issuer-scoped shard of a Certificate Revocation List
  - `BlockedKey` — cryptographic keys prohibited from use in new certificates
  - `Incident` / incident serial tables — records of CA security incidents affecting specific certificate serials
- **Neighbouring context relationships:**
  - **Upstream (external):** ACME clients (subscribers) submit certificate requests via the ACMEv2 HTTP API (port 4001)
  - **Downstream / peer internal services (all gRPC):**
    - `VA` (Validation Authority) — performs domain control validation and CAA checking
    - `CA` (Certificate Authority) — signs precertificates and final certificates, generates OCSP
    - `SA` (Storage Authority) — sole owner of persistent state (MySQL via ProxySQL)
    - `RA` (Registration Authority) — orchestrates the ACME workflow between VA, CA, and SA
    - `Publisher` — submits certificates/precertificates to Certificate Transparency logs
    - `CRLStorer` — uploads signed CRL shards to object storage (AWS S3)
    - `NonceService` — issues and redeems anti-replay nonces
    - `AkamaiPurger` — purges OCSP URLs from CDN cache
  - **External infrastructure dependencies:** Redis (OCSP response cache and rate-limit counters), AWS S3 (CRL storage), Certificate Transparency logs, PKCS#11 HSMs (key storage), Consul (service discovery), Jaeger (distributed tracing)

## Use Cases / User Stories

- **As an ACME client**, I want to create a new account so that I can request certificates. → `RPC NewRegistration`
- **As an ACME client**, I want to update my account contact information or key so that my account stays current. → `RPC UpdateRegistration`
- **As an ACME client**, I want to deactivate my account so that it can no longer be used. → `RPC DeactivateRegistration`
- **As an ACME client**, I want to create a new certificate order for a set of domain names so that I can obtain a certificate. → `RPC NewOrder` / `RPC NewOrderAndAuthzs`
- **As an ACME client**, I want to retrieve pending authorizations and respond to domain control validation challenges so that my domain ownership is confirmed. → `RPC PerformValidation` (RA) → `RPC PerformValidation` (VA), `RPC IsCAAValid`
- **As an ACME client**, I want to finalize an order by submitting a CSR so that a certificate is issued. → `RPC FinalizeOrder` → `RPC IssuePrecertificate` → `RPC IssueCertificateForPrecertificate`
- **As an ACME client**, I want to retrieve my issued certificate by serial so that I can install it. → `RPC GetCertificate`
- **As an ACME client**, I want to revoke a certificate using my account key or the certificate's private key so that it is marked invalid. → `RPC RevokeCertByApplicant`, `RPC RevokeCertByKey`
- **As an ACME client**, I want to deactivate an authorization so that it can no longer be used. → `RPC DeactivateAuthorization`
- **As a CA operator / administrator**, I want to administratively revoke a certificate (e.g. for key compromise) so that relying parties are protected. → `RPC AdministrativelyRevokeCertificate`
- **As a CA operator**, I want to block a specific public key so that certificates with that key cannot be issued. → `RPC AddBlockedKey`, `RPC KeyBlocked`
- **As a relying party / OCSP client**, I want to check the revocation status of a certificate so that I know whether to trust it. → `RPC GenerateOCSP`, `RPC GetCertificateStatus`, `RPC GetRevocationStatus`
- **As a relying party / CRL consumer**, I want to download up-to-date CRL shards so that I have a complete list of revoked certificates. → `RPC GenerateCRL`, `RPC UploadCRL`, `RPC LeaseCRLShard`, `RPC UpdateCRLShard`, `RPC GetRevokedCerts`
- **As the Certificate Transparency ecosystem**, I want precertificates submitted to CT logs before final issuance so that issuance is publicly auditable. → `RPC SubmitToSingleCTWithResult`
- **As a CA operator**, I want OCSP URLs purged from CDN caches after revocation so that up-to-date status is served promptly. → `RPC Purge`
- **As the rate-limiting subsystem**, I want to count recent certificate issuances, orders, registrations, and authorizations per account or IP so that abuse limits can be enforced. → `RPC CountCertificatesByNames`, `RPC CountOrders`, `RPC CountRegistrationsByIP`, `RPC CountRegistrationsByIPRange`, `RPC CountPendingAuthorizations2`, `RPC CountInvalidAuthorizations2`, `RPC CountFQDNSets`
- **As a CA operator**, I want to identify all certificate serials affected by a declared incident so that impacted subscribers can be notified. → `RPC IncidentsForSerial`, `RPC SerialsForIncident`
- **As an ACME client**, I want to obtain an anti-replay nonce for each request so that replayed requests are rejected. → `RPC Nonce`, `RPC Redeem`

## Business Rules

- **Domain control validation (DCV) must precede certificate issuance:** An `Order` can only be finalized (CSR submitted) once all associated `Authorization` objects are in the `valid` state. (inferred from `authz2` status field and `FinalizeOrder`/`PerformValidation` flow)
- **CAA must be checked at or near the time of issuance:** `IsCAAValid` is invoked as part of the validation flow, enforcing CAA DNS record constraints before a certificate can be issued.
- **Precertificate must be submitted to CT logs before final certificate issuance:** `IssuePrecertificate` precedes `IssueCertificateForPrecertificate`; SCTs collected from CT logs are embedded in the final certificate (`IssueCertificateForPrecertificateRequest.SCTs`).
- **Certificate profile integrity is enforced across the RA/CA boundary:** A `certProfileHash` is computed over the certificate profile at precertificate issuance and must match when the final certificate is issued, preventing profile tampering between round-trips.
- **Blocked keys must not be used in new certificates:** `KeyBlocked` is checked against the `blockedKeys` table (keyed by SPKI hash) before issuance proceeds.
- **Rate limits are enforced per account, per IP, per IP range, and per domain:** Counts of certificates, orders, registrations, and authorizations are checked before new resources are created; the `newOrdersRL`, `certificatesPerName`, and Redis-backed counters enforce these limits. A `limitsExempt` flag on `NewOrderRequest` can bypass limits (inferred, likely for testing or internal accounts).
- **Each order is associated with exactly one set of domain names (`fqdnSets`) and one registration:** The `orders`, `orderFqdnSets`, and `orderToAuthz2` tables enforce this relationship.
- **Authorizations expire:** The `authz2.expires` column is indexed and checked; expired authorizations cannot be used to finalize orders.
- **CRL shards are leased before being written:** `LeaseCRLShard` / `UpdateCRLShard` implement an optimistic lock on `crlShards.leasedUntil` to prevent concurrent CRL generation conflicts. (inferred)
- **Revocation reason `keyCompromise` cannot be used for malformed certificates:** If the `malformed` flag is set in `AdministrativelyRevokeCertificateRequest`, `keyCompromise` is disallowed because the key cannot be extracted for blocking.
- **A `replacesSerial` field on `NewOrderRequest` enables ARI (ACME Renewal Information) renewal:** If provided, `ReplacementOrderExists` is used to enforce that only one active replacement order exists per prior certificate serial. (inferred)
- **Registrations and certificates are keyed by JWK public key:** `GetRegistrationByKey` and `GetSerialsByKey` use public key identity; key-based lookups enforce one-account-per-key semantics. (inferred)
- **Incidents are tracked per serial in dedicated per-incident tables:** The `incidents` table references per-incident serial tables (e.g. `incident_foo`, `incident_bar`), allowing targeted mass-revocation workflows.
- **OCSP responses are cached in Redis** to reduce load on the OCSP signing infrastructure; `Purge` ensures CDN-cached responses are invalidated after revocation. (inferred from Redis dependency with `redis-ocsp.config` and `AkamaiPurger`)
- **Serial numbers are tracked independently of certificates:** The `serials` table and `AddSerial` RPC record serial allocation before the certificate DER is stored, enabling auditability even if issuance is interrupted. (inferred)
