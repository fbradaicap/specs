---
repo: fastly-exporter
spec_type: functional
commit: bebe68cfaf4ceab83b844e46ddd566b2408ba213
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 1ebc9d177a62ad41f42f7a653648848917c9f9eef32d8bda8269bd9a92885943
generated_at: 2026-06-30T14:50:59.294173194+02:00
generator: specsync
---

## Business Purpose

`fastly-exporter` is a Prometheus exporter that bridges the Fastly API and Prometheus monitoring. It continuously polls multiple Fastly APIs (Real-time Analytics, Services, POPs/Datacenters, Custom TLS Certificates, Edge Dictionaries, and Product Entitlements) and exposes the collected data as Prometheus metrics on an HTTP `/metrics` endpoint. It exists so that Fastly customers can monitor their CDN services, TLS certificate expiry, datacenter topology, and edge dictionary state using standard Prometheus/Grafana tooling.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Fastly CDN observability / metrics export.
- **Core domain entities / aggregates owned:**
  - `Service` — Fastly service metadata (ID, name, active version).
  - `Datacenter` — Fastly POP metadata (code, name, group, coordinates).
  - `Certificate` — Custom TLS certificate metadata (CN, name, issuer, expiry, serial number).
  - `Dictionary` — Edge dictionary metadata (item count, digest, last-updated timestamp).
  - `Product` — Product entitlement status (e.g., Origin Inspector, Domain Inspector).
  - `RealTimeStats` — Per-service, per-datacenter real-time analytics metrics.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Fastly API (`api.fastly.com`, `rt.fastly.com`) — the sole data source; this service is a pure consumer.
  - **Downstream:** Prometheus scraper — consumes the `/metrics` HTTP endpoint exposed by this service. No events or messages are produced.

## Use Cases / User Stories

- **As a platform engineer**, I want all Fastly services visible to my API token to be automatically discovered and exported as Prometheus metrics, so that I do not have to manually configure each service. _(Service list polling; `ServiceCache.Refresh`)_
- **As a platform engineer**, I want real-time CDN traffic statistics (requests, bandwidth, errors, cache hit rates, etc.) per service and per datacenter exported as Prometheus gauges, so that I can alert on and graph CDN performance. _(Real-time Analytics API polling; `rt` package)_
- **As a security/operations engineer**, I want custom TLS certificate expiry timestamps exposed as Prometheus metrics, so that I can alert before certificates expire. _(Certificate list polling; `CertificateCache.Refresh`; metric `fastly_rt_cert_expiry_timestamp_seconds`)_
- **As a platform engineer**, I want Fastly POP (datacenter) metadata (code, name, group, lat/lon) exposed as Prometheus info metrics, so that I can enrich dashboards with geographic context. _(Datacenter list polling; `DatacenterCache.Refresh`; metric `fastly_rt_datacenter_info`)_
- **As a platform engineer**, I want edge dictionary item counts and last-updated timestamps exported as Prometheus metrics, so that I can monitor dictionary state changes. _(Dictionary info polling; `DictionaryInfoCache.Refresh`; metrics `fastly_rt_dictionary_item_count`, `fastly_rt_dictionary_last_updated_timestamp_seconds`)_
- **As a platform engineer**, I want to filter which services are exported by ID, name allowlist/blocklist regex, or a hash-based shard selector, so that I can run multiple exporter instances without duplication. _(`-service`, `-service-allowlist`, `-service-blocklist`, `-service-shard` flags)_
- **As a platform engineer**, I want to filter which Prometheus metrics are exported by allowlist/blocklist regex, so that I can reduce cardinality. _(`-metric-allowlist`, `-metric-blocklist` flags)_
- **As a platform engineer**, I want to optionally receive only aggregate (non-per-datacenter) real-time stats, so that I can reduce metric cardinality when per-POP granularity is unnecessary. _(`-aggregate-only` flag)_
- **As a platform engineer**, I want the exporter to automatically check product entitlements (Origin Inspector, Domain Inspector), so that only licensed advanced metrics are scraped and exported. _(`ProductCache.Refresh`; `HasAccess`)_

## Business Rules

- A valid Fastly API token must be supplied via `-token` flag or `FASTLY_API_TOKEN` environment variable; the exporter exits with an error if neither is present.
- The `-subsystem` flag is deprecated and will be fixed to `rt` in a future version; the value `origin` is explicitly rejected.
- The `-api-refresh` flag is deprecated in favour of `-service-refresh`.
- Certificate refresh interval, if enabled, must be between 10 minutes and 24 hours; values outside this range are clamped with a warning. A value of `0` disables certificate refresh entirely.
- Datacenter refresh interval must be between 10 minutes and 1 hour; values outside this range are clamped with a warning.
- Service refresh interval must be between 15 seconds and 10 minutes (inferred from README truncation and flag descriptions); values outside this range are clamped.
- Dictionary refresh interval must be between 1 minute and 24 hours (inferred from flag description).
- If the Fastly API returns HTTP 403 (Forbidden) for the certificate endpoint on the first request (when the cache is empty), certificate collection is automatically disabled for the lifetime of the process.
- If the Fastly API returns HTTP 403 for the dictionary endpoint on the first request, dictionary collection is automatically disabled. (inferred, mirrors certificate behaviour)
- The `ProductDefault` product is always considered accessible (`HasAccess` returns `true`) without querying the entitlement API; only `origin_inspector` and `domain_inspector` are checked via the API.
- Certificate serial numbers are converted from decimal to hexadecimal before being emitted as Prometheus label values.
- The digest of edge dictionaries is intentionally **not** exposed as a Prometheus label to avoid cardinality explosion.
- All outbound HTTP requests to Fastly APIs carry a `Fastly-Key` authentication header and a service-specific `User-Agent` header.
- Pagination is followed for the certificates endpoint (up to 200 results per page) and the service list endpoint; all pages are consumed before the cache is updated.
- The Docker image default listen address is `0.0.0.0:8080`; the binary default is `127.0.0.1:8080`.
- The Docker image runs as a non-root `fastly-exporter` user. (inferred from Dockerfile `useradd` + `USER fastly-exporter`)
