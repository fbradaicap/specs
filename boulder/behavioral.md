---
repo: boulder
spec_type: behavioral
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:57:58.750743927+02:00
generator: specsync
---

## API Contracts

Boulder is an ACME certificate authority implementation (Let's Encrypt CA). All internal service communication uses **gRPC over Protocol Buffers**. There is no REST or GraphQL API evidenced in the proto artifacts; the ACMEv2 HTTP interface is exposed externally on port 4001 but its OpenAPI/HTTP route definitions are not included in the provided snapshot.

### Protocol: gRPC (proto3)

---

#### Service: `AkamaiPurger` (`akamai/proto/akamai.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Purge` | `PurgeRequest` | `google.protobuf.Empty` | Purge URLs from Akamai CDN cache |

**Message: `PurgeRequest`**
- `repeated string urls = 1`

---

#### Service: `CertificateAuthority` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `IssuePrecertificate` | `IssueCertificateRequest` | `IssuePrecertificateResponse` | Issue a precertificate (poison-extension cert) |
| `IssueCertificateForPrecertificate` | `IssueCertificateForPrecertificateRequest` | `core.Certificate` | Issue final certificate for an existing precertificate |

**Message: `IssueCertificateRequest`**
- `bytes csr = 1`
- `int64 registrationID = 2`
- `int64 orderID = 3`
- `int64 issuerNameID = 4`
- `string certProfileName = 5`

**Message: `IssuePrecertificateResponse`**
- `bytes DER = 1`
- `bytes certProfileHash = 2`

**Message: `IssueCertificateForPrecertificateRequest`**
- `bytes DER = 1`
- `repeated bytes SCTs = 2`
- `int64 registrationID = 3`
- `int64 orderID = 4`
- `bytes certProfileHash = 5`

---

#### Service: `OCSPGenerator` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `GenerateOCSP` | `GenerateOCSPRequest` | `OCSPResponse` | Generate an OCSP response for a certificate |

**Message: `GenerateOCSPRequest`**
- `string status = 2`
- `int32 reason = 3`
- `google.protobuf.Timestamp revokedAt = 7`
- `string serial = 5`
- `int64 issuerID = 6`

**Message: `OCSPResponse`**
- `bytes response = 1`

---

#### Service: `CRLGenerator` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `GenerateCRL` | `stream GenerateCRLRequest` | `stream GenerateCRLResponse` | Sign a CRL; bidirectional streaming |

**Message: `GenerateCRLRequest`** (oneof `payload`)
- `CRLMetadata metadata = 1` — `int64 issuerNameID`, `google.protobuf.Timestamp thisUpdate`, `int64 shardIdx`
- `core.CRLEntry entry = 2`

**Message: `GenerateCRLResponse`**
- `bytes chunk = 1`

---

#### Service: `CRLStorer` (`crl/storer/proto/storer.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `UploadCRL` | `stream UploadCRLRequest` | `google.protobuf.Empty` | Upload a signed CRL (client streaming) |

**Message: `UploadCRLRequest`** (oneof `payload`)
- `CRLMetadata metadata = 1` — `int64 issuerNameID`, `int64 number`, `int64 shardIdx`
- `bytes crlChunk = 2`

---

#### Service: `Chiller` (`grpc/test_proto/interceptors_test.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Chill` | `Time` | `Time` | Test interceptor: sleep for a given duration |

**Message: `Time`**
- `google.protobuf.Duration duration = 2`

---

#### Service: `NonceService` (`nonce/proto/nonce.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Nonce` | `google.protobuf.Empty` | `NonceMessage` | Generate a new ACME replay-nonce |
| `Redeem` | `NonceMessage` | `ValidMessage` | Validate and redeem a nonce |

**Message: `NonceMessage`**
- `string nonce = 1`

**Message: `ValidMessage`**
- `bool valid = 1`

---

#### Service: `Publisher` (`publisher/proto/publisher.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `SubmitToSingleCTWithResult` | `Request` | `Result` | Submit precert/cert to a CT log and return SCT |

**Message: `Request`**
- `bytes der = 1`
- `string LogURL = 2`
- `string LogPublicKey = 3`
- `SubmissionType kind = 5` — enum: `unknown(0)`, `sct(1)`, `info(2)`, `final(3)`

**Message: `Result`**
- `bytes sct = 1`

---

#### Service: `RegistrationAuthority` (`ra/proto/ra.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `NewRegistration` | `core.Registration` | `core.Registration` | Create a new ACME account registration |
| `UpdateRegistration` | `UpdateRegistrationRequest` | `core.Registration` | Update an existing registration |
| `PerformValidation` | `PerformValidationRequest` | `core.Authorization` | Trigger domain validation |
| `DeactivateRegistration` | `core.Registration` | `google.protobuf.Empty` | Deactivate an account |
| `DeactivateAuthorization` | `core.Authorization` | `google.protobuf.Empty` | Deactivate an authorization |
| `RevokeCertByApplicant` | `RevokeCertByApplicantRequest` | `google.protobuf.Empty` | Applicant-initiated certificate revocation |
| `RevokeCertByKey` | `RevokeCertByKeyRequest` | `google.protobuf.Empty` | Key-based certificate revocation |
| `AdministrativelyRevokeCertificate` | `AdministrativelyRevokeCertificateRequest` | `google.protobuf.Empty` | Admin-initiated revocation |
| `NewOrder` | `NewOrderRequest` | `core.Order` | Create a new certificate order |
| `FinalizeOrder` | `FinalizeOrderRequest` | `core.Order` | Finalize order with a CSR |
| `GenerateOCSP` | `GenerateOCSPRequest` (ra-scoped) | `ca.OCSPResponse` | Generate OCSP from current DB status |

**Message: `UpdateRegistrationRequest`**
- `core.Registration base = 1`
- `core.Registration update = 2`

**Message: `PerformValidationRequest`**
- `core.Authorization authz = 1`
- `int64 challengeIndex = 2`

**Message: `RevokeCertByApplicantRequest`**
- `bytes cert = 1`
- `int64 code = 2`
- `int64 regID = 3`

**Message: `RevokeCertByKeyRequest`**
- `bytes cert = 1`

**Message: `AdministrativelyRevokeCertificateRequest`**
- `bytes cert = 1` (deprecated; ignored)
- `string serial = 4` (required)
- `int64 code = 2`
- `string adminName = 3`
- `bool skipBlockKey = 5`
- `bool malformed = 6`

**Message: `NewOrderRequest`**
- `int64 registrationID = 1`
- `repeated string names = 2`
- `string replacesSerial = 3`
- `bool limitsExempt = 4`
- `string certificateProfileName = 5`

**Message: `FinalizeOrderRequest`** (ra-scoped)
- `core.Order order = 1`
- `bytes csr = 2`

---

#### Service: `StorageAuthorityReadOnly` (`sa/proto/sa.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `CountCertificatesByNames` | `CountCertificatesByNamesRequest` | `CountByNames` | Count certs issued per name set |
| `CountFQDNSets` | `CountFQDNSetsRequest` | `Count` | Count FQDN sets issued |
| `CountInvalidAuthorizations2` | `CountInvalidAuthorizationsRequest` | `Count` | Count invalid authorizations |
| `CountOrders` | `CountOrdersRequest` | `Count` | Count orders |
| `CountPendingAuthorizations2` | `RegistrationID` | `Count` | Count pending authzs for a registration |
| `CountRegistrationsByIP` | `CountRegistrationsByIPRequest` | `Count` | Count registrations from an IP |
| `CountRegistrationsByIPRange` | `CountRegistrationsByIPRequest` | `Count` | Count registrations from an IP range |
| `FQDNSetExists` | `FQDNSetExistsRequest` | `Exists` | Check if an FQDN set exists |
| `FQDNSetTimestampsForWindow` | `CountFQDNSetsRequest` | `Timestamps` | Get timestamps of FQDN set issuances |
| `GetAuthorization2` | `AuthorizationID2` | `core.Authorization` | Fetch authorization by ID |
| `GetAuthorizations2` | `GetAuthorizationsRequest` | `Authorizations` | Fetch multiple authorizations |
| `GetCertificate` | `Serial` | `core.Certificate` | Fetch certificate by serial |
| `GetLintPrecertificate` | `Serial` | `core.Certificate` | Fetch lint precertificate by serial |
| `GetCertificateStatus` | `Serial` | `core.CertificateStatus` | Fetch certificate revocation/OCSP status |
| `GetMaxExpiration` | `google.protobuf.Empty` | `google.protobuf.Timestamp` | Get the maximum certificate expiration |
| `GetOrder` | `OrderRequest` | `core.Order` | Fetch order by ID |
| `GetOrderForNames` | `GetOrderForNamesRequest` | `core.Order` | Fetch order matching a set of names |
| `GetPendingAuthorization2` | `GetPendingAuthorizationRequest` | `core.Authorization` | Fetch a pending authorization |
| `GetRegistration` | `RegistrationID` | `core.Registration` | Fetch registration by ID |
| `GetRegistrationByKey` | `JSONWebKey` | `core.Registration` | Fetch registration by JWK |
| `GetRevocationStatus` | `Serial` | `RevocationStatus` | Fetch revocation status of a serial |
| `GetRevokedCerts` | `GetRevokedCertsRequest` | `stream core.CRLEntry` | Stream revoked cert entries |
| `GetSerialMetadata` | `Serial` | `SerialMetadata` | Fetch metadata for a serial |
| `GetSerialsByAccount` | `RegistrationID` | `stream Serial` | Stream serials for an account |
| `GetSerialsByKey` | `SPKIHash` | `stream Serial` | Stream serials for a key hash |
| `GetValidAuthorizations2` | `GetValidAuthorizationsRequest` | `Authorizations` | Fetch valid authorizations |
| `GetValidOrderAuthorizations2` | `GetValidOrderAuthorizationsRequest` | `Authorizations` | Fetch valid authorizations for an order |
| `IncidentsForSerial` | `Serial` | `Incidents` | Fetch incident records for a serial |
| `KeyBlocked` | `SPKIHash` | `Exists` | Check if a key is blocked |
| `PreviousCertificateExists` | `PreviousCertificateExistsRequest` | `Exists` | Check if a prior cert exists for a name |
| `ReplacementOrderExists` | `Serial` | `Exists` | Check if a replacement order exists for a serial |
| `SerialsForIncident` | `SerialsForIncidentRequest` | `stream IncidentSerial` | Stream serials affected by an incident |

---

#### Service: `StorageAuthority` (`sa/proto/sa.proto`)

Exposes all `StorageAuthorityReadOnly` RPCs (identical signatures) plus the following write operations:

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `AddBlockedKey` | `AddBlockedKeyRequest` | `google.protobuf.Empty` | Block a key by SPKI hash |
| `AddCertificate` | `AddCertificateRequest` | `google.protobuf.Empty` | Store a new certificate |
| `AddPrecertificate` | `AddCertificateRequest` | `google.protobuf.Empty` | Store a new precertificate |
| `SetCertificateStatusReady` | `Serial` | `google.protobuf.Empty` | Mark certificate status as ready |
| `AddSerial` | `AddSerialRequest` | `google.protobuf.Empty` | Record a new serial number |
| `DeactivateAuthorization2` | `AuthorizationID2` | `google.protobuf.Empty` | Deactivate an authorization |
| `DeactivateRegistration` | `RegistrationID` | `google.protobuf.Empty` | Deactivate a registration |
| `FinalizeAuthorization2` | `FinalizeAuthorizationRequest` | `google.protobuf.Empty` | Finalize an authorization |
| `FinalizeOrder` | `FinalizeOrderRequest` | `google.protobuf.Empty` | Finalize an order record |
| `NewOrderAndAuthzs` | `NewOrderAndAuthzsRequest` | `core.Order` | Atomically create order and authorizations |
| `NewRegistration` | `core.Registration` | `core.Registration` | Create a new registration record |
| `RevokeCertificate` | _Not determinable from code._ | `google.protobuf.Empty` | Revoke a certificate |
| `SetOrderError` | _Not determinable from code._ | `google.protobuf.Empty` | Record an error on an order |
| `SetOrderProcessing` | _Not determinable from code._ | `google.protobuf.Empty` | Mark order as processing |
| `UpdateRevokedCertificate` | _Not determinable from code._ | `google.protobuf.Empty` | Update revocation fields |
| `LeaseCRLShard` | _Not determinable from code._ | _Not determinable from code._ | Lease a CRL shard for generation |
| `UpdateCRLShard` | _Not determinable from code._ | `google.protobuf.Empty` | Update CRL shard metadata |

---

#### Service: `VA` (`va/proto/va.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `PerformValidation` | `PerformValidationRequest` | `ValidationResult` | Perform ACME domain challenge validation |

**Message: `PerformValidationRequest`**
- `string domain = 1`
- `core.Challenge challenge = 2`
- `AuthzMeta authz = 3` — `string id`, `int64 regID`

**Message: `ValidationResult`**
- `repeated core.ValidationRecord records = 1`
- `core.ProblemDetails problems = 2`

---

#### Service: `CAA` (`va/proto/va.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `IsCAAValid` | `IsCAAValidRequest` | `IsCAAValidResponse` | Check CAA DNS records for a domain |

**Message: `IsCAAValidRequest`**
- `string domain = 1` (may have wildcard prefix, e.g. `*.example.com`)
- `string validationMethod = 2`
- `int64 accountURIID = 3`

**Message: `IsCAAValidResponse`**
- `core.ProblemDetails problem = 1` (empty if valid)

---

### Shared Core Message Types (`core/proto/core.proto`)

| Message | Key Fields |
|---------|-----------|
| `Certificate` | `int64 registrationID`, `string serial`, `string digest`, `bytes der`, `Timestamp issued`, `Timestamp expires` |
| `CertificateStatus` | `string serial`, `string status`, `Timestamp ocspLastUpdated`, `Timestamp revokedDate`, `int64 revokedReason`, `bool isExpired`, `int64 issuerID` |
| `Registration` | `int64 id`, `bytes key`, `repeated string contact`, `bool contactsPresent`, `string agreement`, `bytes initialIP`, `Timestamp createdAt`, `string status` |
| `Authorization` | `string id`, `string identifier`, `int64 registrationID`, `string status`, `Timestamp expires`, `repeated Challenge challenges` |
| `Order` | `int64 id`, `int64 registrationID`, `Timestamp expires`, `ProblemDetails error`, `string certificateSerial`, `string status`, `repeated string names`, `bool beganProcessing`, `Timestamp created`, `repeated int64 v2Authorizations`, `string certificateProfileName` |
| `Challenge` | `int64 id`, `string type`, `string status`, `string token`, `string keyAuthorization`, `repeated ValidationRecord validationrecords`, `ProblemDetails error`, `Timestamp validated` |
| `CRLEntry` | `string serial`, `int32 reason`, `Timestamp revokedAt` |
| `ProblemDetails` | `string problemType`, `string detail`, `int32 httpStatus` |

---

## Event Schemas

_Not determinable from code._

No message broker, Kafka topics, RabbitMQ queues, or other async event bus integrations are evidenced in the extracted facts or source artifacts. All inter-service communication is synchronous gRPC.

---

## Input / Output Formats

**Serialization:** Protocol Buffers (proto3) for all gRPC service interfaces. The `google.golang.org/protobuf` library (v1.33.0) is used for encoding/decoding.

**Streaming patterns:**
- Server streaming: `GetRevokedCerts` returns `stream core.CRLEntry`; `GetSerialsByAccount` and `GetSerialsByKey` return `stream Serial`; `SerialsForIncident` returns `stream IncidentSerial`.
- Client streaming: `UploadCRL` accepts `stream UploadCRLRequest`.
- Bidirectional streaming: `GenerateCRL` accepts `stream GenerateCRLRequest` and returns `stream GenerateCRLResponse`.

**Timestamps:** All time fields use `google.protobuf.Timestamp` (previously stored as nanosecond integers; several fields have been migrated and old `NS`-suffixed fields are reserved).

**Byte fields:** DER-encoded certificates, CSRs, SCTs, OCSP responses, and JWK keys are transmitted as raw `bytes` fields.

**External HTTP (ACMEv2):** Port 4001 (ACMEv2), port 4002/4003 (OCSP) are exposed per `docker-compose.yml`. The WFE2 (`wfe2/`) component handles ACMEv2 HTTP requests, but the HTTP route/OpenAPI contract details are not included in the provided snapshot. Content-type for ACMEv2 is conventionally `application/jose+json` for request bodies and `application/json` for responses per RFC 8555, but this is not directly evidenced in the snapshot.

**JWK/JOSE:** `gopkg.in/go-jose/go-jose.v2` is used for JSON Web Key and JWS handling (ACME account keys and request signing).

**Pagination:** _Not determinable from code._ Streaming RPCs (e.g. `GetSerialsByAccount`) implicitly paginate via gRPC stream consumption rather than offset/cursor pagination.

---

## Error Handling

**gRPC status codes:** Error propagation follows standard gRPC conventions using `google.golang.org/grpc/status`. Specific code-to-error mappings are not fully enumerable from the proto files alone.

**`ProblemDetails` message:** Used throughout for structured error information within response payloads (not as a transport-level error):
- `string problemType` — ACME problem type URI (e.g. `urn:ietf:params:acme:error:*`)
- `string detail` — human-readable description
- `int32 httpStatus` — HTTP status code equivalent (relevant for ACMEv2 surface)

**Order errors:** `core.Order` contains an embedded `ProblemDetails error` field to communicate order-level failures without requiring a gRPC error status.

**CAA validation:** `IsCAAValidResponse` carries a `core.ProblemDetails problem` field; an empty problem indicates success.

**Validation results:** `ValidationResult` carries a `core.ProblemDetails problems` field for challenge validation failures.

**Revocation reason codes:** `int32 reason` / `int64 code` fields in revocation messages map to RFC 5280 CRL reason codes; specific enumeration is not defined in the proto files (bare integers).

**Key blocking:** `AdministrativelyRevokeCertificateRequest.malformed = true` prevents key blocking when the certificate cannot be parsed; using `keyCompromise` reason is disallowed in that case (enforced at the RA layer).

---

## Versioning

**Proto package namespacing:** Each service has its own Go package path (e.g. `github.com/letsencrypt/boulder/ca/proto`, `github.com/letsencrypt/boulder/sa/proto`). There is no URI-level or header-based versioning; versioning is managed through field reservation (`reserved` field numbers) and additive field additions per proto3 evolution rules.

**Field reservation:** Numerous fields are marked `reserved` with comments indicating previously used field names (e.g., nanosecond timestamp variants replaced by `google.protobuf.Timestamp` equivalents). This pattern provides backward-compatible schema evolution without breaking existing serialized data.

**Service split versioning:** The `StorageAuthority` / `StorageAuthorityReadOnly` split and the `OCSPGenerator` / `CertificateAuthority` / `CRLGenerator` split represent operational versioning — access control boundaries are enforced by deploying separate gRPC server instances with different client allowlists rather than API version strings.

**ACMEv2:** The service targets ACMEv2 (RFC 8555) exclusively; ACMEv1 fields are reserved in several proto messages (e.g., `reserved 7 // previously ACMEv1 combinations` in `Authorization`). No URI versioning suffix is present in the gRPC service definitions.
