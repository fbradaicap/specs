---
repo: hit-counter
spec_type: functional
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:44.326295242+02:00
generator: specsync
---

## Business Purpose
This service implements a page hit counter running on Fastly's edge compute network. It intercepts requests to a configured origin website, records per-page view counts in a Fastly KV Store named `pagehits`, and exposes a statistics endpoint that lists all tracked pages along with their hit totals. It exists to demonstrate and provide edge-native analytics augmentation of a static site without requiring any server-side infrastructure.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge analytics / page-view tracking.
- **Core domain entities / aggregates:**
  - **PageHit** — a keyed counter entry in the `pagehits` KV Store, where the key is a page path and the value is the cumulative hit count.
- **Relationships to neighbouring contexts:**
  - **Upstream:** Any origin website (default: `fastly.github.io/my-site/`) — this service proxies and enriches requests to it.
  - **Downstream:** None detected; the KV Store is internal to this service. The `/stats` route surfaces aggregated data to end-users.

## Use Cases / User Stories

- **As a site visitor**, I want pages I browse to be transparently proxied through the edge so that my experience is unaffected while my visit is silently counted.
  - Evidence: each inbound page request increments the corresponding counter in the `pagehits` KV Store before (or after) being forwarded to the origin backend.
- **As a site owner / analyst**, I want to view a summary page listing all tracked pages and their hit counts so that I can understand which content is most popular.
  - Evidence: README documents a `/stats` (e.g. `/my-site/stats/`) route that returns a synthetic HTML page with the per-page hit data.
- **As a developer**, I want to configure my own origin domain and root path so that I can apply the hit-counter logic to any website.
  - Evidence: README instructs changing `fastly.toml` backend address and the `root` variable in `src/index.js`.
- **As a developer**, I want to deploy the application to Fastly Compute with a single CLI command (`fastly compute publish`) so that the edge service is live without managing servers.

## Business Rules

- Every incoming request for a page under the configured `root` path **must** increment that page's counter in the `pagehits` KV Store by 1. (inferred)
- The `pagehits` KV Store **must** be created and linked to the service before deployment; the store name is fixed as `pagehits`. (inferred from README)
- The `/stats` route **must not** proxy to the origin; it returns a synthetic HTML response generated from KV Store data instead of forwarding the request. (inferred)
- Requests are routed via Expressly; unrecognised routes are presumed to be forwarded to the origin backend. (inferred)
- The origin backend address and the `root` path variable **must** be configured prior to first deployment; changing the backend address after deployment requires use of the Fastly CLI rather than redeployment alone.
- A valid `FASTLY_API_TOKEN` environment variable (or equivalent CLI authentication) is required to publish the service. (inferred from README)
- The `root` variable defaults to `/my-site/`; all hit-counter tracking and stats scoping is relative to this root path. (inferred)
