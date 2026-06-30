---
repo: fastly-js
spec_type: functional
commit: 2ff10a7eb745a90f2f38e13fe4aaa35ca962115e
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 6ba2660cb9f99488ebf2c1aac9e17f0bd432d3a66d76e4928f652363aa6e4357
generated_at: 2026-06-30T14:53:35.703318254+02:00
generator: specsync
---

## Business Purpose

`fastly-js` is an auto-generated JavaScript client library (npm package `fastly`, v15.1.0) that provides a programmatic interface to the Fastly CDN management API. It exists so that JavaScript/Node.js applications and browser-based tooling can perform any operation available in the Fastly management console — including service configuration, cache management, access control, billing, observability, and account administration — without hand-crafting HTTP requests.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Fastly Platform Management — the full administrative surface of the Fastly CDN platform as exposed by `https://api.fastly.com`.
- **Core domain entities / aggregates the library operates on** (inferred from API module names in `src/api/`):
  - **Service & Versioning** — Services, service versions, domains, backends
  - **Traffic Routing & Control** — ACLs, ACL entries, ACLs-in-Compute, conditions, cache settings, apex redirects, VCL
  - **Edge Compute** — Compute@Edge ACLs, config stores, config store items
  - **Security** — WAF, TLS (bulk certs, subscriptions, domains, configurations), purge
  - **Observability** — Real-time analytics, stats, logging endpoints (many providers)
  - **Account & Billing** — Customers, users, billing addresses, invoices (EOM/MTD), usage metrics, automation tokens
- **Relationships to neighbouring contexts:**
  - Downstream consumer of `https://api.fastly.com` (the Fastly platform backend) — this library is purely a client; it owns no data.
  - Upstream of any JavaScript application or tooling that depends on the `fastly` npm package.

## Use Cases / User Stories

- **As a developer**, I want to create, read, update, and delete Fastly **services and service versions** so that I can manage CDN delivery configurations programmatically (`AclApi`, `CacheSettingsApi`, `ConditionApi`, etc. all require `service_id` + `version_id`).
- **As a developer**, I want to manage **ACL entries** (including bulk updates of up to 1,000 entries) attached to a service version so that I can control IP-based access at the edge (`AclApi` → `POST /service/{service_id}/version/{version_id}/acl`; `AclEntryApi` → `PATCH /service/{service_id}/acl/{acl_id}/entries`).
- **As a developer**, I want to create and manage **ACLs in Compute** so that I can enforce access-control logic inside Fastly Compute@Edge functions (`AclsInComputeApi` → `POST /resources/acls`, `DELETE /resources/acls/{acl_id}`).
- **As a developer**, I want to configure **apex redirects** for a service version so that bare-domain traffic is redirected to the `www` subdomain with the desired HTTP status code (`ApexRedirectApi` → `POST /service/{service_id}/version/{version_id}/apex-redirects`).
- **As a developer**, I want to create and revoke **automation tokens** so that CI/CD pipelines can authenticate with the Fastly API without using personal credentials (`AutomationTokensApi` → `POST /automation-tokens`, `GET /automation-tokens/{id}`).
- **As a billing administrator**, I want to retrieve **end-of-month invoices**, **month-to-date invoices**, and **service-level usage metrics** so that I can reconcile CDN spend (`BillingInvoicesApi` → `GET /billing/v3/invoices/{invoice_id}`, `GET /billing/v3/invoices/month-to-date`; `BillingUsageMetricsApi` → `GET /billing/v3/service-usage-metrics`).
- **As an account administrator**, I want to manage the **billing address** associated with a customer account (`BillingAddressApi` → `POST/DELETE/GET /customer/{customer_id}/billing_address`).
- **As a developer**, I want to define **cache settings** (TTL, stale-if-error TTL, VCL fetch action) for a service version so that object freshness behaviour matches business requirements (`CacheSettingsApi` → `POST /service/{service_id}/version/{version_id}/cache_settings`).
- **As a developer**, I want to create and manage **conditions** (VCL conditional expressions with priority ordering) so that rules are applied selectively to requests/responses (`ConditionApi` → `POST /service/{service_id}/version/{version_id}/condition`).
- **As a developer**, I want to create and manage **config stores** and their key-value items so that Compute@Edge functions have access to shared configuration data at runtime (`ConfigStoreApi` → `POST /resources/stores/config`; `ConfigStoreItemApi`).

## Business Rules

- **Authentication is mandatory:** Every API call uses the `token` auth scheme. In Node.js environments, the library will automatically authenticate using the `FASTLY_API_TOKEN` environment variable if present; in browser environments this auto-authentication is skipped (inferred).
- **Service version must be a draft for structural changes:** ACLs and other version-scoped resources (cache settings, conditions, apex redirects) must be attached to a draft (inactive) service version; the version must subsequently be activated to take effect (documented in `AclApi.createAcl` and `deleteAcl` JSDoc).
- **Required path parameters are enforced client-side:** All API methods throw a synchronous `Error` before making any HTTP call if a required parameter (`service_id`, `version_id`, `acl_id`, `customer_id`, `invoice_id`, etc.) is `undefined` or `null`.
- **ACL name format:** ACL names must start with an alphanumeric character and contain only alphanumeric characters, underscores, and whitespace (documented in `AclApi`).
- **Bulk ACL entry update limit:** A single bulk-update request may contain a maximum of 1,000 ACL entries; exceeding this requires contacting Fastly support (`AclEntryApi.bulkUpdateAclEntries` JSDoc).
- **Base path selection:** The default API base URL is `https://api.fastly.com`. Callers may override the server by passing a valid `_base_path_index`; an out-of-range index throws an `Error` client-side.
- **Billing usage pagination:** `getServiceLevelUsage` defaults to a page size of 1,000 results with a maximum of 10,000; cursor-based pagination is used for subsequent pages (`BillingUsageMetricsApi`).
- **`start_month` and `end_month` are required** for `getUsageMetrics`; missing either throws a client-side error (`BillingUsageMetricsApi`).
- **Condition priority:** Condition execution order is determined by a numeric priority value; lower numbers execute first; default priority is `100` (inferred from `ConditionApi` parameter default).
- **Condition type is required:** The `type` field of a condition must be provided; it is documented as "Required" in the JSDoc even though it is not enforced with a pre-flight null-check (inferred from JSDoc comment).
- **Library is auto-generated:** All source files carry the notice `NOTE: This class is auto generated. Do not edit the class manually.` — manual edits will be lost on regeneration.
- **Browser compatibility:** The `fs` Node.js built-in is explicitly excluded in the `browser` field of `package.json`, indicating the library is designed to run in both Node.js and browser environments with appropriate shimming.
