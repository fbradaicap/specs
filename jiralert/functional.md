---
repo: jiralert
spec_type: functional
commit: 3567c32d289b72a6249df61a4f00fbd1018f62e4
model: claude-sonnet-4-6
prompt_version: v1
input_hash: b0230a420a46e50880b88948a4b8d7f32c217c53ec64df989cb4f39d4315723e
generated_at: 2026-06-30T14:54:46.099632837+02:00
generator: specsync
---

## Business Purpose

JIRAlert is a webhook receiver bridge between Prometheus Alertmanager and Atlassian JIRA. It listens for Alertmanager webhook notifications and automatically creates, updates, or reopens JIRA issues based on firing alerts. It exists to give operations teams a traceable, human-actionable ticket trail for infrastructure alerts without requiring manual issue creation.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Alert-to-ticket lifecycle management.
- **Core entities/aggregates:**
  - `Receiver` — a named configuration binding an Alertmanager receiver to a JIRA project and credentials.
  - `Alert` / `AlertGroup` — an Alertmanager notification payload (group of alerts sharing a `group_by` key).
  - `JIRAIssue` — the JIRA issue created or updated in response to an alert group.
- **Relationships:**
  - **Upstream:** Prometheus Alertmanager (sends webhook `POST /alert` payloads to JIRAlert).
  - **Downstream:** Atlassian JIRA REST API (issues are created/updated/reopened via `go-jira`).
  - Exposes Prometheus metrics, making it observable by a Prometheus server (downstream consumer of `/metrics`).

## Use Cases / User Stories

- **As an Alertmanager operator**, I want firing alerts to automatically create JIRA issues so that on-call engineers have a ticket to track and resolve the incident. _(→ `POST /alert`)_
- **As an Alertmanager operator**, I want a single JIRA issue per alert group key (not one per individual alert) so that duplicate tickets are avoided for the same logical problem. _(→ `group_by`-keyed deduplication in `notify.NewReceiver`)_
- **As an on-call engineer**, I want a previously resolved JIRA issue to be automatically reopened when the same alert fires again so that recurring issues are not silently missed. _(→ `reopen_state` config + `--reopen-tickets` flag)_
- **As an on-call engineer**, I want issues marked with a "won't fix" resolution to never be reopened by JIRAlert so that intentionally dismissed alerts are respected. _(→ `wont_fix_resolution` config)_
- **As a platform team**, I want JIRA issue fields (summary, description, priority, custom fields) to be populated from alert labels using Go templates so that tickets are contextually rich without manual editing. _(→ `template` config + `golang.org/x/text`)_
- **As a platform team**, I want JIRAlert to optionally auto-resolve JIRA issues when alerts resolve so that low-noise environments can close tickets automatically. _(→ `auto_resolve` config section)_
- **As an SRE**, I want to inspect the current loaded configuration via HTTP so that I can verify what is active without restarting the service. _(→ `GET /config`)_
- **As an SRE**, I want a health check endpoint so that container orchestration platforms can confirm the service is alive. _(→ `GET /healthz`)_
- **As a monitoring system**, I want Prometheus metrics exposed by JIRAlert so that request success/failure rates can be tracked. _(→ `GET /metrics`)_

## Business Rules

- **One JIRA issue per alert group key (inferred):** Issues are keyed by the Alertmanager `group_by` label set; a new issue is only created if no open issue for that key already exists.
- **Resolved alerts do not close issues by default:** When an alert resolves, JIRAlert does not close the corresponding JIRA issue unless `auto_resolve` is explicitly configured.
- **Resolved issues are reopened on re-fire:** If an alert fires and its JIRA issue exists but is in a resolved state, the issue is transitioned back to `reopen_state`. A valid JIRA transition between the resolved state and `reopen_state` must exist or the reopen will fail.
- **"Won't fix" resolution is honoured:** A JIRA issue carrying the configured `wont_fix_resolution` value will never be reopened by JIRAlert, regardless of alert status.
- **Receiver name must match Alertmanager receiver name:** The `receiver` field in the webhook payload is used to look up the configuration; an unmatched receiver returns HTTP 404 and no issue is created.
- **Authentication is required per receiver:** Each receiver must supply either basic-auth credentials (`user` + `password`) or a Personal Access Token (`personal_access_token`) to authenticate against the JIRA API.
- **JIRA issue description is capped at 32,767 characters by default:** Descriptions exceeding `--max-description-length` (default 32767, matching a known JIRA Server limit) are truncated before submission.
- **JIRA label length guard (inferred):** When `--hash-jira-label` is enabled, alert label key-value pairs inside `JIRALERT{...}` are hashed to prevent exceeding JIRA's 255-character label limit; without this flag the legacy `ALERT{...}` format is used (deprecated).
- **Field updates are individually toggleable:** Summary, description, and priority updates on existing issues can each be independently disabled via `--update-summary`, `--update-description`, and `--update-priority` flags, defaulting to `true`.
- **Retry signalling to Alertmanager:** If the JIRA API returns a retryable error, JIRAlert responds with HTTP 503 to instruct Alertmanager to retry; non-retryable errors return HTTP 400.
- **Configuration supports environment variable substitution (inferred):** Similar to Alertmanager, the YAML configuration file supports `${ENV_VAR}` expansion (referenced in truncated README).
