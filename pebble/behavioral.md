---
repo: pebble
spec_type: behavioral
commit: b37975ee86323dde963d5087b3103d2af34484fd
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9c647701705747ae93296249d39425b915371b7b8aaf4a2a3ebca6fb5d93a790
generated_at: 2026-06-30T14:54:09.090550209+02:00
generator: specsync
---

## API Contracts

Pebble exposes two HTTPS REST APIs (TLS-only; no plain HTTP). Both are governed by the ACME protocol (RFC 8555) and a supplementary management interface. Ports are configurable; defaults observed in `docker-compose.yml` are **14000** (ACME) and **15000** (Management).

### Protocol
HTTPS (TLS) REST/JSON. All ACME endpoints follow the ACME over HTTPS convention; JWS-signed request bodies are used for mutating operations.

---

### ACME API (default port 14000)

Paths are rooted at the `ListenAddress`. The directory path constant is referenced in source as `wfe.DirectoryPath`.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/dir` | ACME Directory — returns URLs for all ACME resources | None | JSON directory object |
| POST | `/sign-me-up` / newAccount | Create or retrieve an ACME account | JWS-signed JSON (contact, termsOfServiceAgreed, externalAccountBinding if EAB required) | JSON account object |
| POST | `/order-plz` / newOrder | Request a new certificate order | JWS-signed JSON (identifiers array) | JSON order object |
| POST | `/authZ/{id}` | Fetch / respond to an authorization | JWS-signed JSON | JSON authorization object |
| POST | `/chalZ/{id}` | Respond to a challenge | JWS-signed JSON (empty `{}` to trigger validation) | JSON challenge object |
| POST | `/finalize-order/{id}` | Finalize an order with a CSR | JWS-signed JSON (csr field, DER base64url) | JSON order object |
| GET/POST | `/certZ/{id}` | Download issued certificate | None / JWS POST-as-GET | PEM certificate chain |
| POST | `/revoke-cert` | Revoke a certificate | JWS-signed JSON (certificate field, DER base64url; optional reason) | HTTP 200 on success |
| POST | `/my-acct/{id}` | Account key rollover / deactivation | JWS-signed JSON | JSON account object |
| HEAD/GET | `/nonce` / newNonce | Obtain a fresh replay-nonce | None | Empty body; `Replay-Nonce` header |

> Exact path strings for all ACME resources are defined as constants in the `wfe` package (e.g., `wfe.DirectoryPath`, `wfe.RootCertPath`). The table above reflects the paths evidenced from source references and ACME RFC 8555 structure as implemented by Pebble.

---

### Management API (default port 15000)

Also HTTPS/TLS. Exposes CA root and intermediate certificates for test trust-anchor installation.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| GET | `/roots/{index}` | Download root CA certificate (index 0 = primary, 1…N = alternates) | None | PEM certificate |
| GET | `/intermediates/{index}` | Download intermediate CA certificate | None | PEM certificate |

Root CA path constant: `wfe.RootCertPath` (e.g., `https://<mgmt-addr><RootCertPath>0`).

---

### Challenge Test Server Management API (`pebble-challtestsrv`, default port 8055)

Plain HTTP REST. All endpoints accept and return JSON. Managed independently from the ACME server.

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|---------|
| POST | `/add-http01` | Add HTTP-01 challenge response | `{"token": "...", "content": "..."}` | HTTP 200 |
| POST | `/del-http01` | Remove HTTP-01 challenge response | `{"token": "..."}` | HTTP 200 |
| POST | `/add-redirect` | Add HTTP redirect rule | `{"from": "...", "to": "..."}` | HTTP 200 |
| POST | `/del-redirect` | Remove HTTP redirect rule | `{"from": "..."}` | HTTP 200 |
| POST | `/set-default-ipv4` | Set default IPv4 for DNS A responses | `{"ip": "..."}` | HTTP 200 |
| POST | `/set-default-ipv6` | Set default IPv6 for DNS AAAA responses | `{"ip": "..."}` | HTTP 200 |
| POST | `/set-txt` | Add DNS-01 TXT record | `{"host": "...", "value": "..."}` | HTTP 200 |
| POST | `/clear-txt` | Remove DNS-01 TXT record | `{"host": "..."}` | HTTP 200 |
| POST | `/add-a` | Add DNS A record | `{"host": "...", "addresses": [...]}` | HTTP 200 |
| POST | `/clear-a` | Remove DNS A record | `{"host": "..."}` | HTTP 200 |
| POST | `/add-aaaa` | Add DNS AAAA record | `{"host": "...", "addresses": [...]}` | HTTP 200 |
| POST | `/clear-aaaa` | Remove DNS AAAA record | `{"host": "..."}` | HTTP 200 |
| POST | `/add-caa` | Add DNS CAA record | `{"host": "...", ...}` | HTTP 200 |
| POST | `/clear-caa` | Remove DNS CAA record | `{"host": "..."}` | HTTP 200 |
| POST | `/set-cname` | Add DNS CNAME record | `{"host": "...", "target": "..."}` | HTTP 200 |
| POST | `/clear-cname` | Remove DNS CNAME record | `{"host": "..."}` | HTTP 200 |
| POST | `/set-servfail` | Force SERVFAIL for a host | `{"host": "..."}` | HTTP 200 |
| POST | `/clear-servfail` | Remove SERVFAIL for a host | `{"host": "..."}` | HTTP 200 |
| POST | `/add-tlsalpn01` | Add TLS-ALPN-01 challenge response | `{"host": "...", "content": "..."}` | HTTP 200 |
| POST | `/del-tlsalpn01` | Remove TLS-ALPN-01 challenge response | `{"host": "..."}` | HTTP 200 |
| POST | `/clear-request-history` | Clear all recorded request history | None / empty | HTTP 200 |
| GET | `/http-request-history` | Retrieve HTTP request history | None | JSON array of request records |
| GET | `/dns-request-history` | Retrieve DNS request history | None | JSON array of request records |
| GET | `/tlsalpn01-request-history` | Retrieve TLS-ALPN-01 request history | None | JSON array of request records |

Exact per-field request/response payload shapes for management endpoints are delegated to the `github.com/letsencrypt/challtestsrv` library and are _Not determinable from code_ in this snapshot beyond what is shown above.

---

## Event Schemas

_Not determinable from code._

No message broker, event queue, or pub/sub mechanism is present. All inter-component communication is synchronous HTTP/HTTPS.

---

## Input / Output Formats

- **Content-Type**: `application/json` for all REST bodies. ACME mutating requests use `application/jose+json` (JWS-wrapped JSON per RFC 8555). Certificate downloads use `application/pem-certificate-chain`.
- **Serialization**: JSON throughout. JWS signing uses the `github.com/go-jose/go-jose/v4` library (JSON serialization). No Protobuf or Avro.
- **ACME Request Body**: Mutating ACME calls carry a JWS compact or JSON serialization envelope with a `payload` (base64url JSON), `protected` header (alg, nonce, url, jwk/kid), and `signature`.
- **Nonce**: Every mutating ACME request must include a fresh `Replay-Nonce` obtained from the `/nonce` endpoint, embedded in the JWS `protected` header. A new nonce is returned in the `Replay-Nonce` response header.
- **Pagination**: _Not determinable from code._
- **Configuration input**: The server is configured via a JSON file (path supplied with `-config` flag). Top-level structure:
  ```json
  {
    "pebble": {
      "listenAddress": "0.0.0.0:14000",
      "managementListenAddress": "0.0.0.0:15000",
      "certificate": "...",
      "privateKey": "...",
      "httpPort": 5002,
      "tlsPort": 5001,
      "ocspResponderURL": "...",
      "externalAccountBindingRequired": false,
      "externalAccountMACKeys": {},
      "domainBlocklist": [],
      "certificateValidityPeriod": 0,
      "retryAfter": { "authz": 0, "order": 0 }
    }
  }
  ```
- **Environment variables** affecting behaviour:
  - `PEBBLE_ALTERNATE_ROOTS` — integer; number of alternate root CAs to generate.
  - `PEBBLE_CHAIN_LENGTH` — integer; intermediate chain length.

---

## Error Handling

The error model follows RFC 8555 ACME problem documents:

- **Error format**: `application/problem+json` response bodies, conforming to RFC 7807, with fields `type` (ACME error URN, e.g., `urn:ietf:params:acme:error:*`), `detail` (human-readable string), and `status` (HTTP status code integer).
- **Common status codes evidenced**:
  - `400 Bad Request` — malformed JWS, invalid nonce, bad CSR, unsupported identifier type, domain on blocklist.
  - `401 Unauthorized` — JWS signature verification failure, account not found.
  - `403 Forbidden` — EAB required but not provided or invalid; account deactivated.
  - `404 Not Found` — unknown order, authorization, challenge, or certificate ID.
  - `405 Method Not Allowed` — wrong HTTP method for endpoint.
  - `409 Conflict` — account already exists (on newAccount with `onlyReturnExisting`).
  - `422 Unprocessable Entity` — validation errors (e.g., identifier rejected).
  - `500 Internal Server Error` — unexpected CA/VA failures.
- **Replay-Nonce errors**: A missing or already-used nonce returns `400` with ACME error type `urn:ietf:params:acme:error:badNonce`.
- **Strict mode** (`-strict` flag): Enables additional validation that rejects certain requests which would otherwise be tolerated, testing stricter RFC compliance; specific behaviours are _Not fully determinable from code_ in this snapshot.
- **Retry-After**: Configurable `RetryAfter.Authz` and `RetryAfter.Order` values (seconds) are sent as `Retry-After` headers on pending authz/order responses.
- **Challenge test server management API**: Returns HTTP `200` on success; error details are _Not determinable from code._

---

## Versioning

- **Module version**: The Go module is `github.com/letsencrypt/pebble/v2`, indicating a **major version 2** via Go module path convention.
- **ACME protocol version**: Implements ACME RFC 8555 (ACMEv2). There is no URI-level version segment on ACME endpoints (e.g., no `/v1/`, `/v2/` prefix); the version is implied by protocol compliance.
- **Binary version**: Embedded at build time via `-ldflags` overriding the `version` variable; exposed via the `-version` CLI flag (prints to stdout, not via an API endpoint).
- **Header-based or schema-evolution versioning**: _Not determinable from code._
