---
repo: hit-counter
spec_type: functional
commit: 8f82dfcca2f12d9460d2444092f6adc817d0afc5
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 40a68b7dd1ff3de50a10db8798da6bbe3cf22a3b2260e27cf990cd550d39f66f
generated_at: 2026-06-30T14:56:25.386423828+02:00
generator: specsync
---

## Business Purpose

This service provides a **page hit counter** capability for websites, deployed as a Fastly Compute edge application. It intercepts incoming HTTP requests to a configured origin site, increments a per-page visit count stored in a Fastly KV Store, and exposes a statistics endpoint that reports cumulative hit counts per page. It exists to add lightweight, edge-native analytics to any static or dynamic website without backend infrastructure changes.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Edge analytics / page traffic tracking.
- **Core entities / aggregates:**
  - **PageHit** — a keyed counter entry in the `pagehits` KV Store, representing the cumulative hit count for a specific URL path.
- **Relationships to neighbouring contexts:**
  - Upstream: an origin website backend (default: `fastly.github.io/my-site/`) — the hit counter proxies all page requests to it.
  - No downstream consumers detected; the stats page is a human-facing read surface, not an event or API feed.

## Use Cases / User Stories

- **As a website visitor**, I want to browse pages on the origin site so that my visit is transparently proxied and counted without any change to my experience.
- **As a site owner**, I want each page request to automatically increment that page's hit count in the `pagehits` KV Store so that I have an accurate record of page popularity.
- **As a site owner**, I want to view a `/stats` page listing all tracked URLs and their cumulative hit counts so that I can understand traffic patterns across my site.
- **As a developer**, I want to configure a custom origin domain and root path (via `fastly.toml` and `src/index.js`) so that I can apply the hit counter to my own website.

## Business Rules

- Every incoming page request **must** increment the corresponding entry in the `pagehits` KV Store by 1 before (or while) proxying to the origin. (inferred)
- The KV Store is named exactly `pagehits`; this name is fixed and referenced in configuration.
- The stats endpoint is served at the path `<root>/stats/` (default: `/my-site/stats/`) and returns a synthetic HTML page — it is **not** proxied to the origin.
- The `root` variable in `src/index.js` defines the base path prefix; requests outside this root are not subject to hit counting. (inferred)
- The origin backend address must be set **before first deployment**; changing it post-deployment requires use of the Fastly CLI directly, not just a code change.
- No authentication or access control is defined for the `/stats` endpoint — it is publicly accessible. (inferred)
- Hit counts are persisted in a KV Store (not in-memory), ensuring counts survive edge node restarts and are durable across requests. (inferred)
