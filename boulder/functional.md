---
repo: boulder
spec_type: functional
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:58:58.292168945+02:00
generator: specsync
---

## Business Purpose

Boulder is the certificate authority (CA) software that implements the ACME (Automatic Certificate Management Environment) protocol, powering Let's Encrypt and operated/forked by Fastly. It provides automated issuance, renewal, and revocation of TLS/SSL X.509 certificates by orchestrating domain validation, order management, certificate signing, and certificate status publication (OCSP, CRL, Certificate Transparency). The service exists to enable any domain owner to obtain trusted certificates programmatically without manual intervention.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** ACME Certificate Authority — encompassing the full lifecycle of TLS certificate issuance under the ACME protocol.
- **Core domain entities / aggregates:**
  - `Registration` — an ACME account (subscriber identity, public key, contact)
  - `Authorization` (`authz2`) — a domain control validation record linking a registration to an identifier
  - `Order` / `OrderToAuthz2` — an issuance request grouping names and associated authorizations
  - `Certificate` / `Precertificate` — issued X.509 leaf certificates and their CT-log precursor
  - `CertificateStatus` — OCSP/revocation state for each issued certificate
  - `CRLShard` — a shard of a Certificate Revocation List
  - `BlockedKey` / `KeyHashToSerial` — key abuse / compromise tracking
  - `Incident` — security/compliance incident records cross-referenced to certificate serials
  - `Nonce` — anti-replay tokens for ACME requests
- **Relationships to neighbouring contexts:**
  - **Upstream (external):** ACME clients (subscribers) interact via the WFE (Web Front End) over HTTPS/ACMEv2 (port 4001).
  - **Downstream / infrastructure:** Certificate Transparency logs (via `Publisher`/`SubmitToSingleCTWithResult`); Akamai CDN cache purge (via `AkamaiPurger`); AWS S3 (CRL storage via `CRLStorer`); Redis (OCSP response cache, rate-limit counters); MariaDB/ProxySQL (persistent state); PKCS#11 HSMs (signing keys).
  - **Internal sub-contexts:** RA (Registration Authority), CA (Certificate Authority), VA (Validation Authority), SA (Storage Authority), OCSP/CRL generators, Publisher, Nonce service — all communicating via internal gRPC.

## Use Cases / User Stories

- **As an ACME client**, I want to create a new account registration so that I can authenticate future certificate requests. → `RPC NewRegistration`, `RPC GetRegistration`, `RPC GetRegistrationByKey`
- **As an ACME client**, I want to update my account contacts or key so that my registration stays current. → `RPC UpdateRegistration`
- **As an ACME client**, I want to deactivate my account so that it can no longer be used to issue certificates. → `RPC DeactivateRegistration`
- **As an ACME client**, I want to create a new certificate order for a set of domain names so that I can begin the issuance process. → `RPC NewOrder`, `RPC NewOrderAndAuthzs`, `RPC GetOrder`, `RPC GetOrderForNames`
- **As an ACME client**, I want to obtain domain control validation challenges so that I can prove ownership of the requested names. → `RPC GetAuthorization2`, `RPC GetAuthorizations2`, `RPC GetPendingAuthorization2`
- **As the RA**, I want to trigger domain validation (HTTP-01, DNS-01, TLS-ALPN-01) so that authorizations can be marked valid or invalid. → `RPC PerformValidation` (RA→VA), `RPC IsCAAValid` (VA CAA service), `RPC FinalizeAuthorization2`
- **As an ACME client**, I want to deactivate a specific authorization so that it is no longer usable. → `RPC DeactivateAuthorization`, `RPC DeactivateAuthorization2`
- **As an ACME client**, I want to finalize an order by submitting a CSR so that a certificate is issued. → `RPC FinalizeOrder`
- **As the CA**, I want to issue a precertificate and submit it to Certificate Transparency logs before issuing the final certificate, so that issuance is publicly auditable. → `RPC IssuePrecertificate`, `RPC SubmitToSingleCTWithResult`, `RPC IssueCertificateForPrecertificate`
- **As the SA**, I want to store issued certificates and precertificates so that they can be retrieved and their status tracked. → `RPC AddCertificate`, `RPC AddPrecertificate`, `RPC AddSerial`, `RPC SetCertificateStatusReady`
- **As a relying party or OCSP responder**, I want to retrieve the current revocation status of a certificate so that TLS clients can verify its validity. → `RPC GetCertificateStatus`, `RPC GetRevocationStatus`, `RPC GenerateOCSP` (CA OCSPGenerator)
- **As an ACME client**, I want to revoke a certificate using my account key or the certificate's private key so that compromised certificates are promptly invalidated. → `RPC RevokeCertByApplicant`, `RPC RevokeCertByKey`
- **As a CA administrator**, I want to administratively revoke a certificate (e.g. for policy reasons or key compromise) so that the certificate is invalidated regardless of subscriber action. → `RPC AdministrativelyRevokeCertificate`
- **As the CRL pipeline**, I want to generate and publish sharded CRLs so that revocation information is available via the CRL distribution points. → `RPC GenerateCRL`, `RPC LeaseCRLShard`, `RPC UpdateCRLShard`, `RPC UploadCRL`, `RPC GetRevokedCerts`
- **As the rate-limiting subsystem**, I want to count recent registrations, orders, certificates, and authorizations per account or IP so that abuse limits can be enforced. → `RPC CountRegistrationsByIP`, `RPC CountRegistrationsByIPRange`, `RPC CountCertificatesByNames`, `RPC CountOrders`, `RPC CountPendingAuthorizations2`, `RPC CountInvalidAuthorizations2`, `RPC CountFQDNSets`
- **As the WFE**, I want to issue and redeem anti-replay nonces so that ACME requests cannot be replayed. → `RPC Nonce`, `RPC Redeem`
- **As a security operator**, I want to block a compromised public key so that no further certificates are issued for it. → `RPC AddBlockedKey`, `RPC KeyBlocked`
- **As a security operator**, I want to look up all certificates affected by a declared incident so that remediation (e.g. mass revocation) can be performed. → `RPC IncidentsForSerial`, `RPC SerialsForIncident`
- **As the CDN/OCSP infrastructure**, I want to purge cached OCSP or CRL URLs from Akamai so that revocation updates propagate promptly. → `RPC Purge` (AkamaiPurger)
- **As an ACME client**, I want to replace an existing certificate order using the `replaces` field so that renewals are tracked for rate-limit exemption purposes. → `RPC NewOrder` (`replacesSerial`), `RPC ReplacementOrderExists`

## Business Rules

- **Domain validation is required before issuance:** An order cannot be finalized unless all associated authorizations are in `valid` status; authorizations transition through `pending` → `valid` or `invalid`. (inferred from `authz2` schema `status` field and `FinalizeAuthorization2`/`FinalizeOrder` flow)
- **CAA must be checked at issuance time:** The VA's `IsCAAValid` RPC must be invoked per domain before a certificate is issued to verify Certification Authority Authorization DNS records.
- **Certificate Transparency submission precedes final certificate issuance:** A precertificate must be submitted to at least one CT log (`SubmitToSingleCTWithResult`) and SCTs collected before `IssueCertificateForPrecertificate` is called. (inferred from `IssueCertificateForPrecertificateRequest.SCTs` field)
- **Certificate profile integrity must be preserved across roundtrips:** The `certProfileHash` is computed over the profile at precertificate issuance and verified again at final issuance to ensure the profile has not changed. (from `IssuePrecertificateResponse.certProfileHash` and `IssueCertificateForPrecertificateRequest.certProfileHash`)
- **Blocked keys cannot receive new certificates:** `RPC KeyBlocked` is checked against the `blockedKeys` table; if a key is blocked, issuance must be refused. (inferred from `blockedKeys` table and `KeyBlocked` RPC)
- **Revocation reason `keyCompromise` requires a blockable key:** If the `malformed` flag is set on `AdministrativelyRevokeCertificateRequest`, `keyCompromise` reason is disallowed because the key cannot be blocked.
- **Key compromise revocation blocks the key for future issuance:** `AdministrativelyRevokeCertificate` with `skipBlockKey=false` and reason `keyCompromise` results in the subject key being added to `blockedKeys`. (inferred from `skipBlockKey` field and `AddBlockedKey` RPC)
- **Rate limits are enforced per registration, IP, IP range, and FQDN set:** Multiple count RPCs (`CountRegistrationsByIP`, `CountCertificatesByNames`, `CountOrders`, etc.) feed the rate-limiting subsystem; the `newOrdersRL` and `certificatesPerName` tables back these counters.
- **Orders have an expiry:** The `orders.expires` column and `Order.expires` proto field enforce a deadline; orders past expiry are invalid. (inferred from schema)
- **Authorizations have an expiry:** `authz2.expires` enforces a time-bound on domain control proofs; expired authorizations cannot be used. (inferred from schema)
- **An order can reference a replacement for rate-limit purposes:** `NewOrderRequest.replacesSerial` and `ReplacementOrderExists` allow orders to be identified as renewals, which may affect rate-limit exemptions. (inferred)
- **OCSP responses are cached in Redis and must be purged on revocation:** The Redis dependency (OCSP config) and `AkamaiPurger.Purge` together enforce that revocation state is propagated to CDN-cached OCSP responses. (inferred from infrastructure config)
- **CRL shards are leased before generation to prevent concurrent writes:** `LeaseCRLShard` / `UpdateCRLShard` and the `crlShards.leasedUntil` column implement an optimistic lock preventing two generators from writing the same shard simultaneously. (inferred from schema)
- **Incident tables are separate schemas, one table per incident:** The `incidents_sa` database holds per-incident serial tables (`incident_foo`, `incident_bar`), cross-referenced via the `incidents` registry table's `serialTable` column.
- **Registrations require a valid JWK:** `GetRegistrationByKey` looks up accounts by JSON Web Key; registration stores the public key (`registrations.jwk`). (inferred)
- **Orders undergoing processing are locked:** `SetOrderProcessing` transitions an order to a processing state before the CA begins signing, preventing duplicate issuance. (inferred from `beganProcessing` column and `SetOrderProcessing` RPC)
- **`limitsExempt` flag allows bypassing rate limits for specific orders:** `NewOrderRequest.limitsExempt` is present; its authorization gating is `_Not determinable from code._` at this level but the field implies privileged issuance paths. (inferred)
