---
repo: fastly-js
spec_type: behavioral
commit: 2ff10a7eb745a90f2f38e13fe4aaa35ca962115e
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 6ba2660cb9f99488ebf2c1aac9e17f0bd432d3a66d76e4928f652363aa6e4357
generated_at: 2026-06-30T14:53:35.703318254+02:00
generator: specsync
---

## API Contracts

This library is a **REST** client SDK (auto-generated from OpenAPI document version 1.0.0) that wraps the Fastly public API. The single upstream base URL is `https://api.fastly.com`. All methods return Promises and accept an `options` object. Authentication is performed via a bearer/API token (`FASTLY_API_TOKEN` environment variable or explicit `apiClient.authenticate()` call), mapped to the auth scheme named `token`.

The table below documents every endpoint group surfaced in the sampled source. Because the repository contains ~100 API class files (999 source files, one class per resource area), only the groups evidenced in the snapshot are listed; the pattern is consistent across all classes.

### ACL (`AclApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/service/{service_id}/version/{version_id}/acl` | Create a new ACL on a service version | Form: `name` (string, optional) | `AclResponse` |
| `DELETE` | `/service/{service_id}/version/{version_id}/acl/{acl_name}` | Delete an ACL from a service version | Path params only | `InlineResponse200` |
| `GET` | `/service/{service_id}/version/{version_id}/acl/{acl_name}` | Describe an ACL | Path params only | `AclResponse` |
| `GET` | `/service/{service_id}/version/{version_id}/acl` | List ACLs for a service version | Path params only | `[AclResponse]` |
| `PUT` | `/service/{service_id}/version/{version_id}/acl/{acl_name}` | Update an ACL | Form: `name` | `AclResponse` |

### ACL Entries (`AclEntryApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `PATCH` | `/service/{service_id}/acl/{acl_id}/entries` | Bulk-update ACL entries (max 1000) | JSON body: `BulkUpdateAclEntriesRequest` | `InlineResponse200` |
| `POST` | `/service/{service_id}/acl/{acl_id}/entry` | Add a single ACL entry | JSON body: `AclEntry` | `AclEntryResponse` |
| `DELETE` | `/service/{service_id}/acl/{acl_id}/entry/{acl_entry_id}` | Delete an ACL entry | Path params only | `InlineResponse200` |
| `GET` | `/service/{service_id}/acl/{acl_id}/entry/{acl_entry_id}` | Describe an ACL entry | Path params only | `AclEntryResponse` |
| `GET` | `/service/{service_id}/acl/{acl_id}/entries` | List ACL entries | Path + query params | `[AclEntryResponse]` |
| `PATCH` | `/service/{service_id}/acl/{acl_id}/entry/{acl_entry_id}` | Update an ACL entry | JSON body: `AclEntry` | `AclEntryResponse` |

### ACLs in Compute (`AclsInComputeApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/resources/acls` | Create a new compute ACL | JSON body: `ComputeAclCreateAclsRequest` | `ComputeAclCreateAclsResponse` |
| `DELETE` | `/resources/acls/{acl_id}` | Delete a compute ACL | Path params only | _(empty)_ |
| `GET` | `/resources/acls/{acl_id}/entries` | List entries of a compute ACL | Path + query params | `ComputeAclListEntries` |
| `GET` | `/resources/acls/{acl_id}/entry/{acl_entry_id}` | Lookup a single compute ACL entry | Path params only | `ComputeAclLookup` |
| `GET` | `/resources/acls` | List all compute ACLs | _(none)_ | `ComputeAclList` |
| `PATCH` | `/resources/acls/{acl_id}/entries` | Update compute ACL entries | JSON body: `ComputeAclUpdate` | _(empty)_ |

### Apex Redirects (`ApexRedirectApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/service/{service_id}/version/{version_id}/apex-redirects` | Create apex redirect | Form: `service_id`, `version`, `created_at`, `deleted_at`, `updated_at`, `status_code`, `domains` (csv), `feature_revision` | `ApexRedirect` |
| `DELETE` | `/service/{service_id}/version/{version_id}/apex-redirects/{apex_redirect_id}` | Delete apex redirect | Path params only | `InlineResponse200` |
| `GET` | `/service/{service_id}/version/{version_id}/apex-redirects/{apex_redirect_id}` | Get apex redirect | Path params only | `ApexRedirect` |
| `GET` | `/service/{service_id}/version/{version_id}/apex-redirects` | List apex redirects | Path params only | `[ApexRedirect]` |
| `PUT` | `/apex-redirects/{apex_redirect_id}` | Update apex redirect | Form fields as above | `ApexRedirect` |

### Automation Tokens (`AutomationTokensApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/automation-tokens` | Create an automation token | JSON body: `AutomationTokenCreateRequest` | `AutomationTokenCreateResponse` |
| `GET` | `/automation-tokens/{id}` | Retrieve automation token by ID | Path param `id` | `AutomationTokenResponse` |
| `GET` | `/automation-tokens` | List automation tokens | Query params (pagination) | `[AutomationTokenResponse]` |
| `DELETE` | `/automation-tokens/{id}` | Revoke an automation token | Path param `id` | `AutomationTokenErrorResponse` |
| `GET` | `/automation-tokens/{id}/services` | List services for an automation token | Path param `id` + query params | `InlineResponse2004` |

### Billing Address (`BillingAddressApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/customer/{customer_id}/billing_address` | Add billing address | JSON/vnd.api+json body: `BillingAddressRequest` | `BillingAddressResponse` |
| `DELETE` | `/customer/{customer_id}/billing_address` | Delete billing address | Path param only | _(empty)_ |
| `GET` | `/customer/{customer_id}/billing_address` | Get billing address | Path param only | `BillingAddressResponse` |
| `PATCH` | `/customer/{customer_id}/billing_address` | Update billing address | JSON/vnd.api+json body: `UpdateBillingAddressRequest` | `BillingAddressResponse` |

### Billing Invoices (`BillingInvoicesApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` | `/billing/v3/invoices/{invoice_id}` | Get invoice by ID | Path param `invoice_id` (Number) | `EomInvoiceResponse` |
| `GET` | `/billing/v3/invoices/month-to-date` | Get month-to-date invoice | _(none)_ | `MtdInvoiceResponse` |
| `GET` | `/billing/v3/invoices` | List invoices | Query params (sort, pagination) | `ListEomInvoicesResponse` |

### Billing Usage Metrics (`BillingUsageMetricsApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `GET` | `/billing/v3/service-usage-metrics` | Get service-level usage | Query: `product_id`, `service`, `usage_type_name`, `start_month`, `end_month`, `limit` (default `1000`, max `10000`), `cursor` | `Serviceusagemetrics` |
| `GET` | `/billing/v3/usage-metrics` | Get monthly customer usage metrics | Query: `start_month` (required), `end_month` (required) | `Usagemetric` |

### Cache Settings (`CacheSettingsApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/service/{service_id}/version/{version_id}/cache_settings` | Create cache settings object | Form: `action`, `cache_condition`, `name`, `stale_ttl`, `ttl` | `CacheSettingResponse` |
| `DELETE` | `/service/{service_id}/version/{version_id}/cache_settings/{cache_settings_name}` | Delete cache settings object | Path params only | `InlineResponse200` |
| `GET` | `/service/{service_id}/version/{version_id}/cache_settings/{cache_settings_name}` | Get a cache settings object | Path params only | `CacheSettingResponse` |
| `GET` | `/service/{service_id}/version/{version_id}/cache_settings` | List cache settings objects | Path params only | `[CacheSettingResponse]` |
| `PUT` | `/service/{service_id}/version/{version_id}/cache_settings/{cache_settings_name}` | Update cache settings object | Form fields as above | `CacheSettingResponse` |

### Conditions (`ConditionApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/service/{service_id}/version/{version_id}/condition` | Create condition | Form: `comment`, `name`, `priority` (default `'100'`), `statement`, `service_id`, `version`, `type` | `ConditionResponse` |
| `DELETE` | `/service/{service_id}/version/{version_id}/condition/{condition_name}` | Delete condition | Path params only | `InlineResponse200` |
| `GET` | `/service/{service_id}/version/{version_id}/condition/{condition_name}` | Get condition | Path params only | `ConditionResponse` |
| `GET` | `/service/{service_id}/version/{version_id}/condition` | List conditions | Path params only | `[ConditionResponse]` |
| `PUT` | `/service/{service_id}/version/{version_id}/condition/{condition_name}` | Update condition | Form fields as above | `ConditionResponse` |

### Config Store (`ConfigStoreApi`)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/resources/stores/config` | Create config store | Form: `name` | `ConfigStoreResponse` |
| `DELETE` | `/resources/stores/config/{config_store_id}` | Delete config store | Path param only | `InlineResponse200` |
| `GET` | `/resources/stores/config/{config_store_id}` | Get config store | Path param only | `ConfigStoreResponse` |
| `GET` | `/resources/stores/config` | List config stores | _(none)_ | `[ConfigStoreResponse]` |
| `PUT` | `/resources/stores/config/{config_store_id}` | Update config store | Form: `name` | `ConfigStoreResponse` |
| `GET` | `/resources/stores/config/{config_store_id}/info` | Get config store metadata | Path param only | `ConfigStoreInfoResponse` |

> **Note:** The source tree contains ~100 additional API classes (e.g. `BackendApi`, `DomainApi`, `ServiceApi`, `TlsApi`, `WafApi`, `LoggingApi`, etc.) following the identical structural pattern. Complete endpoint details for those classes are not fully reproduced here as the source samples were truncated, but the protocol, base URL, authentication, and serialisation conventions are uniform across all of them.

---

## Event Schemas

_Not determinable from code._

No message broker integration, event topics, or asynchronous publish/subscribe patterns are present in the source. This library is a purely synchronous REST client SDK.

---

## Input / Output Formats

**Protocol:** HTTP/1.1 REST over TLS to `https://api.fastly.com`.

**HTTP client:** `superagent` ^6.1.0 (wrapped by `ApiClient`).

**Request serialisation** — varies by endpoint and is explicitly set per call:

| Content-Type | Used for |
|---|---|
| `application/x-www-form-urlencoded` | Most create/update operations on service-version-scoped resources (ACL, CacheSettings, Condition, ConfigStore, etc.) |
| `application/json` | Bulk operations and resource-level endpoints (BulkUpdateAclEntries, AclEntry create/update, AclsInCompute, AutomationTokens, BillingAddress via `application/vnd.api+json`) |
| `application/vnd.api+json` | Billing address add/update |
| _(no body)_ | DELETE and most GET requests |

**Response serialisation:** `application/json` on all endpoints; some endpoints additionally accept `application/problem+json` (e.g. `GET /automation-tokens/{id}`).

**Array parameters:** Collection parameters (e.g. `domains` in ApexRedirect) are serialised as CSV via `apiClient.buildCollectionParam(value, 'csv')`.

**Pagination:** Cursor-based pagination is implemented on at least the following endpoints:
- `GET /billing/v3/service-usage-metrics` — query params `limit` (default `'1000'`, max 10 000) and `cursor` (value from `next_cursor` field of the previous response; empty string for the first page).
- `GET /automation-tokens/{id}/services` — query-param pagination evidenced by `InlineResponse2004` response type.

**Date fields:** ISO 8601 format (e.g. `created_at`, `deleted_at`, `updated_at` in ApexRedirect).

**Base path selection:** Each API class stores `basePaths` as an array; callers may pass `_base_path_index` in the options object to select an alternate server entry.

---

## Error Handling

**Client-side (SDK) validation** — before any HTTP call is dispatched, each `*WithHttpInfo` method checks required parameters and throws a plain JavaScript `Error` synchronously:

```
throw new Error("Missing the required parameter 'service_id'.");
```

Missing required path parameters (`service_id`, `version_id`, `acl_id`, `acl_name`, `customer_id`, `id`, `invoice_id`, `start_month`, `end_month`, etc.) and out-of-range `_base_path_index` values all trigger this guard.

**HTTP error responses** — the `superagent`-backed `ApiClient` propagates HTTP errors as Promise rejections. The response models that represent error payloads evidenced in the source are:

| Model | Used by |
|---|---|
| `AutomationTokenErrorResponse` | `DELETE /automation-tokens/{id}` |
| `BillingAddressVerificationErrorResponse` | `POST /customer/{customer_id}/billing_address`, `PATCH` equivalent |
| `Error` (generic) | `BillingInvoicesApi`, `BillingUsageMetricsApi` |

Some GET endpoints declare `application/problem+json` as an accepted response content-type (e.g. `GET /automation-tokens/{id}`), indicating RFC 7807 problem detail objects may be returned by the upstream API for error cases.

**Specific HTTP status code mappings** are not enumerated in the client source; the library is auto-generated and delegates status code interpretation to `superagent` and the calling application.

---

## Versioning

**OpenAPI document version:** `1.0.0` (declared in every source file header).

**Library version:** `15.1.0` (declared in `package.json` and in every API class JSDoc as `@version 15.1.0`).

**URI versioning** is used selectively by the upstream Fastly API:
- Billing endpoints use `/billing/v3/…` URI prefixes.
- All other resource endpoints (services, ACLs, conditions, etc.) use non-versioned paths (e.g. `/service/{service_id}/version/{version_id}/…`), where `version_id` is a service configuration version number, not an API version.

**No header-based or content-negotiation versioning** is evidenced in the client code.

**Schema evolution:** The library is declared auto-generated ("Do not edit the class manually") from an OpenAPI specification; evolution is managed upstream via the OpenAPI document and reflected in the npm package semver (`15.1.0`).
