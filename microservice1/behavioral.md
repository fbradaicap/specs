---
repo: microservice1
spec_type: behavioral
commit: fc1d6e6323f710ab16190abe9550bf81c2b36f23
model: anthropic:claude-sonnet-4-6
prompt_version: v1
input_hash: d184f820c48574690a1a274d278aa6d784a7a22a7d9a0daa5ad2cff964993cfa
generated_at: 2026-06-30T17:52:55.707749678+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST/HTTP (Jakarta REST via Quarkus REST extension). The service listens on port `8080`.

### Calculator Endpoints

Base path: `/calculator`

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/calculator/add` | Add two numbers | Query params: `a` (double), `b` (double) | `CalculationResult` JSON |
| GET | `/calculator/subtract` | Subtract two numbers | Query params: `a` (double), `b` (double) | `CalculationResult` JSON |
| GET | `/calculator/multiply` | Multiply two numbers | Query params: `a` (double), `b` (double) | `CalculationResult` JSON |
| GET | `/calculator/divide` | Divide two numbers | Query params: `a` (double), `b` (double) | `CalculationResult` JSON |

**`CalculationResult` response shape:**
```json
{
  "result": <double>,
  "operation": "<string>",
  "error": "<string | null>"
}
```

### Hello Endpoint

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/hello` | Returns a greeting | None | `text/plain` or `application/json` string |

> Note: Two `@GET` methods are declared on `/hello` with different `@Produces` types (`text/plain` and `application/json`). Content negotiation via the `Accept` header determines which is served. The plain-text variant returns `"Hello"`; the JSON variant returns the string `"2"`.

### Weather Endpoints

Base path: `/weather`

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/weather` | Get current weather for a city | Query param: `city` (string) | `WeatherInfo` JSON |
| GET | `/weather/forecast` | Get multi-day forecast for a city | Query params: `city` (string), `days` (int, 1–7) | `WeatherForecast` JSON |

**`WeatherInfo` response shape:**
```json
{
  "city": "<string>",
  "temperature": <double>,
  "condition": "<string>",
  "humidity": <int>,
  "error": "<string | null>"
}
```

**`WeatherForecast` response shape:**
```json
{
  "city": "<string>",
  "days": <int>,
  "forecast": [
    {
      "day": "<string>",
      "temperature": <double>,
      "condition": "<string>",
      "humidity": <int>
    }
  ],
  "error": "<string | null>"
}
```

---

## Event Schemas

_Not determinable from code._

---

## Input / Output Formats

- **Serialization:** JSON (`application/json`) via Jackson (`quarkus-rest-jackson`, `quarkus-rest-client-jackson`) for all endpoints except `GET /hello` which also supports `text/plain`.
- **Request parameters:** All inputs are passed as URL query parameters (`@QueryParam`); there are no request bodies defined.
- **Response envelopes:** No wrapper envelope; responses are flat JSON objects (`CalculationResult`, `WeatherInfo`, `WeatherForecast`) serialized directly from POJO fields. Public fields are serialized as-is (no custom serializer is evidenced).
- **Pagination:** Not applicable; no paginated responses are present.
- **Content negotiation:** `GET /hello` declares two `@GET` handlers—one producing `text/plain` and one `application/json`—resolved by the client `Accept` header.

---

## Error Handling

Error handling is application-level (inline in resource methods), not via a global exception mapper. No HTTP error status codes other than `200` are evidenced in the source.

| Endpoint | Condition | Behaviour |
|----------|-----------|-----------|
| `GET /calculator/divide` | `b == 0` | Returns HTTP `200` with `CalculationResult { result: 0, operation: "divide", error: "Error: Division by zero" }` |
| `GET /weather` | `city` param absent or empty | Returns HTTP `200` with `WeatherInfo { error: "Error: city parameter is required" }` |
| `GET /weather/forecast` | `city` param absent or empty | Returns HTTP `200` with `WeatherForecast { error: "Error: city parameter is required" }` |
| `GET /weather/forecast` | `days` ≤ 0 or > 7 | Returns HTTP `200` with `WeatherForecast { error: "Error: days must be between 1 and 7" }` |

- **Error payload structure:** Errors are embedded in the normal response object via a nullable `error` (string) field. On success, `error` is `null`.
- **HTTP status codes:** Only `200 OK` is explicitly evidenced (confirmed by the test assertion `statusCode(200)`). No `4xx`/`5xx` mappings are declared in the source.
- **Validation:** Input validation is performed manually within method bodies; there is no Bean Validation (`@NotNull`, `@Valid`, etc.) or JAX-RS `ExceptionMapper` present in the snapshot.

---

## Versioning

No API versioning strategy is evidenced in the code. There are no URI version prefixes (e.g., `/v1/`), version headers, or schema evolution mechanisms present in the snapshot.
