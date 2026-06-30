---
repo: jiralert
spec_type: behavioral
commit: 3567c32d289b72a6249df61a4f00fbd1018f62e4
model: claude-sonnet-4-6
prompt_version: v1
input_hash: b0230a420a46e50880b88948a4b8d7f32c217c53ec64df989cb4f39d4315723e
generated_at: 2026-06-30T14:54:46.099632837+02:00
generator: specsync
---

## API Contracts

JIRAlert exposes a synchronous HTTP/REST interface. It listens on `:9097` by default (configurable via `-listen-address`).

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| `POST` | `/alert` | Receive Alertmanager webhook notifications and create/update/reopen JIRA issues | JSON body: Alertmanager `Data` object (see schema below) | `200 OK` on success; `400 Bad Request` on decode failure or non-retryable notifier error; `404 Not Found` if receiver name is unknown; `500 Internal Server Error` on JIRA client init failure; `503 Service Unavailable` on retryable notifier error |
| `GET` | `/` | Home/index page | None | HTML index page |
| `GET` | `/config` | Dump current loaded configuration | None | Configuration representation |
| `GET` | `/healthz` | Health check | None | `200 OK` with body `OK` |
| `GET` | `/metrics` | Prometheus metrics exposition | None | Prometheus text/protobuf metrics |
| `GET` | `/debug/pprof/*` | Go pprof profiling endpoints (imported via `net/http/pprof`) | None | pprof output |

**Alertmanager webhook payload structure** (as decoded into `alertmanager.Data`):

```
{
  "receiver": "<string>",         // Alertmanager receiver name; must match a configured JIRAlert receiver
  "status":   "<string>",         // e.g. "firing" or "resolved"
  "alerts": [
    {
      "status": "<string>",
      "labels": { "<key>": "<value>", ... }
      // additional Alertmanager alert fields per https://prometheus.io/docs/alerting/configuration/#webhook_config
    }
  ],
  "groupLabels":       { "<key>": "<value>", ... },
  "commonLabels":      { "<key>": "<value>", ... },   // present per Alertmanager spec
  "commonAnnotations": { "<key>": "<value>", ... },   // present per Alertmanager spec
  "externalURL":       "<string>"                     // present per Alertmanager spec
}
```

Protocol: **HTTP/1.1** (plain HTTP; TLS termination not evidenced in code).

---

## Event Schemas

_Not determinable from code._

JIRAlert does not produce or consume any message-queue or streaming events. It communicates synchronously: inbound via HTTP webhook from Alertmanager, and outbound via the JIRA REST API (using `github.com/andygrunwald/go-jira`).

---

## Input / Output Formats

**Content type:**
- Inbound (`POST /alert`): `application/json` — the request body is decoded with `encoding/json`.
- `/metrics`: Prometheus exposition format (text or protobuf, content-negotiated by `promhttp.Handler`).
- All other endpoints: plain text or HTML (no structured format evidenced).

**Serialization:** JSON for the webhook endpoint; YAML for the configuration file (`gopkg.in/yaml.v3`).

**Pagination:** _Not determinable from code._ No pagination is applied to the webhook endpoint.

**Request envelope:** None — the raw Alertmanager `Data` struct is decoded directly from the request body without a wrapper envelope.

**Response envelope:** None — success responses return HTTP `200` with no structured body; error responses use `http.Error` (plain text body with the error message and an appropriate HTTP status code).

---

## Error Handling

Error responses are produced by an `errorHandler` function that calls `http.Error`, emitting a plain-text body and one of the following status codes:

| Condition | HTTP Status |
|-----------|-------------|
| Request body JSON decode failure | `400 Bad Request` |
| Receiver name not found in configuration | `404 Not Found` |
| JIRA client initialisation failure | `500 Internal Server Error` |
| Notifier error flagged as retryable (instructs Alertmanager to retry) | `503 Service Unavailable` |
| Notifier error flagged as non-retryable | `400 Bad Request` |
| Health check | `200 OK` (body: `OK`) |

The `503` response is explicitly intended to signal Alertmanager's retry logic. The `400` used for non-retryable notifier errors is acknowledged in the source as inaccurate but is used to prevent Alertmanager from retrying.

Prometheus counter `requestTotal` is incremented with label `"200"` on success, and error labels are recorded per `errorHandler`. Structured errors are logged via `go-kit/log` at appropriate levels (`error`, `debug`).

Validation behaviour: receiver name lookup is performed before JIRA client construction; if `conf.User`/`conf.Password` and `conf.PersonalAccessToken` are all absent, no JIRA client is created (behaviour in that case is _not determinable from code_ beyond the implicit nil-client path).

---

## Versioning

No URI versioning, header versioning, or schema evolution strategy is evidenced in the code. All endpoints are unversioned (e.g., `/alert`, `/healthz`). The build version string is injected at compile time via `-ldflags "-X main.Version=$(VERSION)"` and logged at startup, but this does not influence the API contract. _No API versioning strategy is applied._
