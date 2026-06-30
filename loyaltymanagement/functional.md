---
repo: loyaltymanagement
spec_type: functional
commit: a0321cd4386bc946a394b585316e19a23f985828
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 469b02de1c019b90b692b2f0f3c1092f8f3229e17169b313f078fb1f6acdcbef
generated_at: 2026-06-30T15:59:26.665857914+02:00
generator: specsync
---

## Business Purpose

This service, owned by Carrefour (`com.carrefour`), is intended to manage customer loyalty programmes — likely encompassing the earning, redemption, and tracking of loyalty points or rewards for Carrefour customers. It is structured as a standalone Spring Boot REST API with persistence, suggesting it will serve as the authoritative backend for loyalty-related operations. The service is currently in an early scaffold/prototype state (version `0.0.1-SNAPSHOT`) with no implemented business logic beyond the application entry point.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Loyalty Management — responsible for the lifecycle of customer loyalty accounts, points balances, and reward transactions within the Carrefour retail ecosystem.
- **Core domain entities / aggregates:** _Not determinable from code._ No entity classes, JPA models, or DB schema have been committed yet; persistence infrastructure (Spring Data JPA, H2) is declared but unused.
- **Neighbouring contexts:** The `com.carrefour` group ID places this alongside other Carrefour services (e.g. order management, customer profile), which are likely upstream providers of purchase events or customer identity. _Specific relationships are not determinable from code._

## Use Cases / User Stories

No controllers, endpoints, or business logic have been implemented in the committed source. The following use cases are inferred from the service name, group ID, and declared technology stack:

- As a **customer**, I want my loyalty points balance tracked so that I can redeem rewards on future purchases. _(inferred; no endpoint implemented)_
- As a **point-of-sale system**, I want to post purchase transactions to the loyalty service so that points are credited automatically. _(inferred; no endpoint implemented)_
- As a **customer**, I want to redeem accumulated points against a purchase so that I receive a discount or reward. _(inferred; no endpoint implemented)_
- As a **back-office operator**, I want to view and manage loyalty accounts via a REST API documented through OpenAPI/Swagger UI (`springdoc-openapi`) so that support and administration tasks can be performed. _(inferred from `springdoc-openapi-starter-webmvc-ui` dependency; no endpoint implemented)_

## Business Rules

- _Not determinable from code._ No domain logic, validation, or constraint code has been committed. The following structural observations hold:
  - The service uses an **in-memory H2 database** (runtime scope), indicating it is not yet configured for production-grade persistence; data does not survive restarts. _(inferred — likely a development/hackathon convenience, not an intended production constraint)_
  - The application context must load successfully with zero configuration beyond `spring.application.name`; this is enforced by the existing `contextLoads` test.
  - Lombok is excluded from the final Spring Boot fat-jar, meaning all Lombok-generated code must be fully resolved at compile time. _(inferred from Maven plugin exclusion configuration)_
