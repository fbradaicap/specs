---
repo: fastly-exporter
spec_type: behavioral
commit: bebe68cfaf4ceab83b844e46ddd566b2408ba213
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 1ebc9d177a62ad41f42f7a653648848917c9f9eef32d8bda8269bd9a92885943
generated_at: 2026-06-30T14:50:59.294173194+02:00
generator: specsync
---

## API Contracts

### Protocol: HTTP (outbound) + Prometheus HTTP exposition (inbound)

This service operates in two distinct interface directions:

1. **Outbound (consumed):** The exporter calls several Fastly REST APIs to collect data.
2. **Inbound (served):** The exporter exposes a Prometheus metrics HTTP endpoint for scraping.

---

#### Inbound: Prometheus Metrics Endpoint

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| GET | `/metrics` | Prometheus metric scrape endpoint | No body; standard Prometheus scrape request | Prometheus text/OpenMetrics exposition format |

- Default listen address: `127.0.0.1:8080` (configurable via `-listen` flag or `FASTLY_EXPORTER_LISTEN` env var).
- The Docker image exposes port `8080` and binds to `0.0.0.0:8080`.

---

#### Outbound: Fastly APIs Consumed

| Method | Path (base: `https://api.fastly.com`) | Purpose | Auth Header | Response Shape |
|--------|--------------------------------------|---------|-------------|---------------|
| GET | `/tls/certificates?page[number]=1&page[size]=200&sort=created_at` | Fetch paginated list of custom TLS certificates | `Fastly-Key: <token>` | `{"data": [{...}], "links": {...}}` |
| GET | `/datacenters` | Fetch list of POPs/datacenters | `Fastly-Key: <token>` | `[{"code":..., "name":..., "group":..., "coordinates":{...}}, ...]` |
| GET | `/entitled-products/<product>` | Check product entitlement (e.g., `origin_inspector`, `domain_inspector`) | `Fastly-Key: <token>` | `{"product":{"id":...}, "has_access": bool, ...}` |
| GET | `/service` (service list) | Fetch list of Fastly services | `Fastly-Key: <token>` | Array of service objects |
| GET | `/service/<id>/version/<ver>/dictionary` | List dictionaries for a service version | `Fastly-Key: <token>` | Array of dictionary objects |
| GET | `/service/<id>/version/<ver>/dictionary/<dict_id>/info` | Fetch dictionary info (item count, digest, last updated) | `Fastly-Key: <token>` | `{"digest":..., "item_count":..., "last_updated":...}` |
| GET | `https://rt.fastly.com/...` | Real-time analytics streaming | `Fastly-Key: <token>` | Real-time stats JSON |

All outbound requests set `Accept: application/json`.

---

#### Prometheus Metrics Exposed

The following metric names are constructed using the configurable `namespace` (default: `fastly`) and subsystem (fixed: `rt`):

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `<ns>_<sub>_cert_expiry_timestamp_seconds` | Gauge | `cn`, `name`, `id`, `issuer`, `sn` | TLS certificate expiry as Unix timestamp |
| `<ns>_<sub>_datacenter_info` | Gauge | `datacenter`, `name`, `group`, `latitude`, `longitude` | Metadata about Fastly POPs |
| `<ns>_<sub>_dictionary_item_count` | Gauge | `service_id`, `service_name`, `version`, `dictionary_id`, `dictionary_name` | Number of items in an edge dictionary |
| `<ns>_<sub>_dictionary_last_updated_timestamp_seconds` | Gauge | `service_id`, `service_name`, `version`, `dictionary_id`, `dictionary_name` | Unix timestamp of last dictionary update |

Additional real-time analytics metrics (from the RT API) are also exposed; their exact names are _Not determinable from code_ based on the sampled files.

---

## Event Schemas

_Not determinable from code._

No message broker, event queue, or pub/sub system is used. All data flows are synchronous HTTP polling.

---

## Input / Output Formats

### Inbound (Prometheus scrape)
- **Content-Type:** Prometheus text exposition format (`text/plain; version=0.0.4`) and/or OpenMetrics (`application/openmetrics-text`), negotiated via `Accept` header by the Prometheus `client_golang` library.
- **Serialization:** Prometheus text format or Protobuf (handled transparently by `prometheus/client_golang`).
- **Pagination:** Not applicable.

### Outbound (Fastly API requests)
- **Content-Type / Accept:** `application/json` on all requests.
- **Serialization:** JSON, decoded via `encoding/json` and `github.com/json-iterator/go`.
- **Pagination:** Cursor-based pagination via `Link` header (`rel="next"`) following RFC 5988 for service and certificate list endpoints. The certificate endpoint also accepts `page[number]` and `page[size]` query parameters (max page size: 200). The `GetNextLink` helper parses `Link` response headers to follow paginated results.
- **Authentication:** Bearer-style token passed as the `Fastly-Key` request header on all outbound calls.
- **User-Agent:** A custom `User-Agent` header is injected on all outbound HTTP requests via `userAgentTransport`.

### Certificate API response envelope
```json
{
  "data": [
    {
      "id": "<string>",
      "attributes": {
        "issued_to": "<string>",
        "name": "<string>",
        "issuer": "<string>",
        "not_after": "2006-01-02T15:04:05.000Z",
        "serial_number": "<decimal string>"
      }
    }
  ],
  "links": {
    "next": "<url or null>",
    "self": "<url>",
    "first": "<url>",
    "last": "<url>"
  }
}
```

### Datacenter API response envelope
```json
[
  {
    "code": "<string>",
    "name": "<string>",
    "group": "<string>",
    "coordinates": {
      "latitude": 0.0,
      "longitude": 0.0
    }
  }
]
```

### Product entitlement response envelope
```json
{
  "product": { "id": "<string>" },
  "has_access": true
}
```

### Dictionary info response envelope
```json
{
  "digest": "<string>",
  "item_count": 0,
  "last_updated": "<RFC3339 or 'YYYY-MM-DD HH:MM:SS'>"
}
```

---

## Error Handling

### Outbound API errors
- **Error type:** `api.Error` struct — `{ Code int, Msg string }`.
- **Construction:** On any non-`200 OK` response from Fastly APIs, `NewError(resp)` is called, which reads `resp.StatusCode` and attempts to JSON-decode `{"msg": "..."}` from the response body.
- **Error string format:** `"api.fastly.com responded with <StatusText> (<Msg>)"` where `<Msg>` is omitted if empty.
- **Special case — 403 Forbidden on certificate endpoint:** If the initial certificate refresh returns HTTP 403 and the local cache is empty (first run), the `CertificateCache` disables itself (`enabled = false`) and subsequent `Refresh` calls become no-ops. This allows the exporter to run without TLS Management permissions.
- **Special case — 403 Forbidden on dictionary endpoint:** Similarly, if a 403 is received and no dictionaries are cached, the `DictionaryInfoCache` disables itself.
- **Refresh interval clamping:** Configuration values outside allowed ranges are silently clamped and a warning is logged rather than treating them as hard errors:
  - `certificate-refresh`: clamped to `[10m, 24h]`; `0` disables it.
  - `datacenter-refresh`: clamped to `[10m, 1h]`.
  - `product-refresh`: clamped to `[10m, 24h]`.
  - `service-refresh`: clamped to `[15s, 10m]`.
  - `dictionary-refresh`: clamped to `[1m, 24h]`.
- **Missing token:** If neither `-token` flag nor `FASTLY_API_TOKEN` environment variable is set, the process logs an error and exits with code `1`.

### Inbound HTTP errors
- _Not determinable from code_ — the Prometheus `/metrics` endpoint error handling is managed by `prometheus/client_golang` internally.

### Logging
- Structured log output in `logfmt` format via `go-kit/log`.
- Log levels: `error`, `warn`, `info`, `debug`. Debug logging enabled via `-debug` flag.

---

## Versioning

- **URI versioning:** Not used on the exporter's own HTTP surface.
- **Binary versioning:** The build embeds a version string via `-ldflags "-X main.programVersion=$VERSION"` and also sets `github.com/prometheus/common/version.Version`. The `-version` flag prints `fastly-exporter v<version>` and exits.
- **Default version:** `"dev"` when built without ldflags injection.
- **Fastly API versioning:** The exporter targets unversioned or implicitly versioned Fastly API paths (e.g., `/tls/certificates`, `/datacenters`). Service-version-scoped paths include the active version number as a path segment (e.g., `/service/<id>/version/<ver>/dictionary`).
- **No API version header or Accept-version strategy** is evident in the outbound requests.
- **Deprecated flags:** `-subsystem` (will be fixed to `rt`), `-api-refresh` (replaced by `-service-refresh`) — both emit deprecation warnings via the logger rather than hard errors.
