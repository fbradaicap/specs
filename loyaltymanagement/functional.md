---
repo: loyaltymanagement
spec_type: functional
commit: a0321cd4386bc946a394b585316e19a23f985828
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: 223517456ad62115d22dfd391894f67014ce1b9e9a6d6184885c09a215615c83
generated_at: 2026-06-30T17:38:11.525463360+02:00
generator: specsync
---

## Business Purpose

This service provides a **loyalty management** capability for Carrefour, intended to manage customer loyalty programmes (e.g. points accrual, redemption, tier management). It is built as a Spring Boot REST API backed by a JPA data store, and exposes an OpenAPI/Swagger UI for API discovery. The service appears to be at an early/skeleton stage — core business logic has not yet been committed to the repository.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Loyalty Management (group ID `com.carrefour`, package `com.carrefour.loyalty`)
- **Core domain entities / aggregates:** _Not determinable from code._ (no entity classes are present in the snapshot)
- **Neighbouring contexts:** The group ID `com.carrefour` indicates this is one microservice within a wider Carrefour retail platform; upstream/downstream relationships (e.g. to a customer identity service, an order/transaction service, or a rewards catalogue) are _not determinable from code._

## Use Cases / User Stories

Because no controllers, entities, or service classes are present in the snapshot, specific use cases cannot be grounded in implemented code. Based on the service name, groupId, and selected dependencies the following are directionally indicated but unconfirmed:

- _Not determinable from code._ — No endpoints, domain models, or business logic have been committed. The repository contains only an application bootstrap class and a context-load test.

## Business Rules

- _Not determinable from code._ — No controllers, entities, validators, or service logic are present; no business rules or invariants can be extracted.
- (inferred) The service uses an **in-memory H2 database** (runtime scope, H2 console dependency included), suggesting it is currently running in a development/prototype configuration rather than against a persistent store.
- (inferred) The service exposes a **Swagger UI** (`springdoc-openapi-starter-webmvc-ui`), implying the API is intended to be self-documenting and likely consumed by other internal services or a frontend.
- (inferred) Java 21 and Spring Boot 4.x are targeted, indicating a modern, long-term-support runtime baseline for the service.
