---
repo: microservice1
spec_type: functional
commit: fc1d6e6323f710ab16190abe9550bf81c2b36f23
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: d184f820c48574690a1a274d278aa6d784a7a22a7d9a0daa5ad2cff964993cfa
generated_at: 2026-06-30T17:52:55.707749678+02:00
generator: specsync
---

## Business Purpose

This service is a demo/utility Quarkus REST microservice that exposes three independent capabilities: a basic arithmetic calculator, a greeting endpoint, and a mock weather information service. It exists primarily as a hackathon demonstration or starter project, providing simple HTTP APIs with no persistent state or external integrations. The weather data is explicitly mocked (hardcoded values), indicating the service is not production-grade for that feature.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Utility / Demo Services — no single cohesive business domain is owned; the service bundles unrelated concerns (arithmetic, greeting, weather).
- **Core domain entities / aggregates:**
  - `CalculationResult` — represents the result of an arithmetic operation (operands are implicit via query params; result, operation name, and optional error are returned).
  - `WeatherInfo` — represents current weather for a city (city, temperature, condition, humidity).
  - `WeatherForecast` / `DayForecast` — represents a multi-day weather forecast per city.
- **Relationships to neighbouring contexts:** The `quarkus-rest-client` / `quarkus-rest-client-jackson` dependencies are present, indicating the service is structured to call external REST APIs (e.g., a real weather API), but no such client is wired in the current code. No upstream or downstream service relationships are determinable from code.

## Use Cases / User Stories

- **As a caller, I want to add two numbers** so that I receive their sum — `GET /calculator/add?a={a}&b={b}`.
- **As a caller, I want to subtract two numbers** so that I receive their difference — `GET /calculator/subtract?a={a}&b={b}`.
- **As a caller, I want to multiply two numbers** so that I receive their product — `GET /calculator/multiply?a={a}&b={b}`.
- **As a caller, I want to divide two numbers** so that I receive their quotient, with division-by-zero guarded — `GET /calculator/divide?a={a}&b={b}`.
- **As a caller, I want a greeting response** so that I can verify the service is alive — `GET /hello`.
- **As a caller, I want current weather information for a named city** so that I can display weather conditions — `GET /weather?city={city}`.
- **As a caller, I want a multi-day weather forecast for a named city** so that I can plan ahead — `GET /weather/forecast?city={city}&days={days}`.

## Business Rules

- **Division by zero protection:** When the divisor (`b`) equals `0` on `GET /calculator/divide`, the service returns a result of `0` with an error message `"Error: Division by zero"` rather than throwing an exception.
- **City parameter is required for weather:** `GET /weather` and `GET /weather/forecast` both require a non-null, non-empty `city` query parameter; missing or blank values return an error field `"Error: city parameter is required"` with a `200` HTTP status (inferred — error is embedded in response body, not an HTTP error code).
- **Forecast days must be between 1 and 7:** `GET /weather/forecast` rejects `days <= 0` or `days > 7` with `"Error: days must be between 1 and 7"` (same inline-error pattern).
- **All calculator endpoints accept `double` operands** via query parameters `a` and `b`; both are required by the method signature — missing parameters will default to `0.0` in Java (inferred).
- **Weather data is mock/static:** Current weather responses are hardcoded for Paris, London, New York, and Tokyo; any other city returns a default `20.0°C / "Unknown"` response rather than a real lookup.
- **All responses are JSON** (via `@Produces(MediaType.APPLICATION_JSON)`) except `GET /hello`, which returns `text/plain`.
- **No authentication or authorisation controls** are present in the codebase.
