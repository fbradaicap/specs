---
repo: pebble
spec_type: functional
commit: b37975ee86323dde963d5087b3103d2af34484fd
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9c647701705747ae93296249d39425b915371b7b8aaf4a2a3ebca6fb5d93a790
generated_at: 2026-06-30T14:54:09.090550209+02:00
generator: specsync
---

## Business Purpose

Pebble is a lightweight, test-focused ACME (Automatic Certificate Management Environment) server implementing the RFC 8555 protocol, originally developed by Let's Encrypt. It exists to give ACME client developers and CI pipelines a realistic but intentionally minimal CA they can run locally or in containers without depending on a production CA. It also ships a companion tool (`pebble-challtestsrv`) that simulates the DNS, HTTP, and TLS infrastructure needed to complete ACME domain-validation challenges.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** ACME Certificate Issuance (test/development environment).
- **Core domain entities / aggregates:**
  - **Account** – ACME account registration, including External Account Binding (EAB).
  - **Order** – certificate order lifecycle (pending → ready → processing → valid/invalid).
  - **Authorization** – domain authorization tied to an order.
  - **Challenge** – individual validation challenge (HTTP-01, DNS-01, TLS-ALPN-01).
  - **Certificate** – issued leaf certificate with configurable chain length and validity period.
  - **CA / Root / Intermediate** – in-memory certificate authority with optional alternate roots.
  - **Blocked Domain** – domain-level policy list preventing issuance.
- **Relationships to neighbouring contexts:**
  - **Upstream (ACME clients):** Any RFC 8555-compliant client (e.g., Certbot, acme.sh) drives Pebble through its ACME API.
  - **Downstream (challenge validation):** `pebble-challtestsrv` acts as a controllable DNS/HTTP/TLS server that Pebble's Validation Authority (VA) queries to verify challenges.
  - **Downstream (OCSP):** An optional external OCSP responder URL can be embedded in issued certificates.

## Use Cases / User Stories

- **As an ACME client developer**, I want to register a new ACME account (with or without External Account Binding) so that I can begin requesting certificates against a local test CA. _(ACME `newAccount` endpoint; EAB enforced by `ExternalAccountBindingRequired` config flag)_
- **As an ACME client developer**, I want to submit a certificate order for one or more domain names so that I can test the full order lifecycle without contacting a production CA. _(ACME `newOrder` endpoint)_
- **As an ACME client developer**, I want to complete HTTP-01, DNS-01, or TLS-ALPN-01 domain validation challenges so that my client's challenge-response logic is exercised end-to-end. _(VA component + `challtestsrv` challenge endpoints)_
- **As a CI pipeline**, I want to retrieve the Pebble root CA certificate from the management API so that I can add it to a test trust store and validate issued certificates. _(Management API `RootCertPath` endpoint, e.g. `https://<mgmt>/roots/0`)_
- **As a test author**, I want to programmatically add and remove DNS records, HTTP challenge responses, and TLS-ALPN certificates via the `challtestsrv` management HTTP API so that I can simulate both passing and failing validations. _(`/add-http01`, `/set-txt`, `/add-a`, `/add-tlsalpn01`, etc.)_
- **As an ACME client developer**, I want to run Pebble in strict mode so that my client is tested against upcoming breaking API changes before they reach production. _(`-strict` flag on the `pebble` binary)_
- **As a CA operator (test)**, I want to configure a domain blocklist so that certain domain names are always rejected during order creation. _(`DomainBlocklist` config field + `db.AddBlockedDomain`)_
- **As a test author**, I want to inspect HTTP, DNS, and TLS-ALPN request history via the `challtestsrv` management API so that I can assert which validation requests were made. _(`/http-request-history`, `/dns-request-history`, `/tlsalpn01-request-history`)_
- **As an ACME client developer**, I want to control the number of alternate root CAs and intermediate chain length via environment variables so that I can test chain-building edge cases. _(`PEBBLE_ALTERNATE_ROOTS`, `PEBBLE_CHAIN_LENGTH` env vars)_

## Business Rules

- **EAB enforcement:** When `ExternalAccountBindingRequired` is `true`, every `newAccount` request must include a valid External Account Binding MAC signed with a pre-configured key; requests without it are rejected.
- **EAB key registry:** Only MAC keys explicitly listed in `ExternalAccountMACKeys` (config) are accepted for EAB; unknown key IDs are rejected.
- **Domain blocklist:** Any domain present in `DomainBlocklist` cannot be included in a certificate order; orders containing blocked domains are denied at creation time.
- **Certificate validity period:** Issued certificates have a configurable validity period (`CertificateValidityPeriod`); the default behavior when unset is _not determinable from code_.
- **Retry-After policy:** The server enforces configurable `RetryAfter` delays for authorization (`Authz`) and order (`Order`) objects, instructing clients how long to wait before polling.
- **TLS-only API:** Both the ACME API and the Management API are served exclusively over HTTPS (`ListenAndServeTLS`); no plain-HTTP ACME surface is exposed.
- **In-memory state only:** All ACME objects (accounts, orders, authorizations, certificates) are stored in an in-memory store (`db.NewMemoryStore`); state is lost on restart. (inferred)
- **Chain length ≥ 1:** The `chainLength` value defaults to 1 and is only overridden if `PEBBLE_CHAIN_LENGTH` parses to a non-negative integer; a value of 0 is technically allowed by the parse logic but may yield degenerate chains. (inferred)
- **Alternate roots ≥ 0:** `PEBBLE_ALTERNATE_ROOTS` defaults to 0; only non-negative parsed values are accepted.
- **Challenge types supported:** HTTP-01, DNS-01, and TLS-ALPN-01 are all supported challenge types; the VA component uses a configurable DNS resolver address for DNS-01 lookups.
- **Strict mode:** When `-strict` is enabled, the WFE applies additional validation corresponding to upcoming RFC/API changes; specific checks are _not determinable from code_ without inspecting the full `wfe` package.
- **Management API is optional:** If `ManagementListenAddress` is empty, the management interface (including root CA certificate retrieval) is disabled entirely.
