---
repo: boulder
spec_type: behavioral
commit: 3e2d852f3cad888de7e0a3c253937290493912bc
model: claude-sonnet-4-6
prompt_version: v1
input_hash: 9cabeedf70a1ffdf15eca52687004aeb1f345306a3d18ec04a399e24757b49a2
generated_at: 2026-06-30T14:58:58.292168945+02:00
generator: specsync
---

## API Contracts

Boulder exposes exclusively **gRPC** (proto3) interfaces. There are no REST/HTTP or GraphQL APIs in the service itself; the ACMEv2 HTTP endpoints (port 4001) are served by the `wfe2` component within the same monorepo. All internal service-to-service communication uses gRPC over mTLS.

### Protocol: gRPC / proto3

---

#### Service: `AkamaiPurger` (`akamai/proto/akamai.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Purge` | `PurgeRequest { repeated string urls }` | `google.protobuf.Empty` | Purge a list of URLs from Akamai CDN cache |

---

#### Service: `CertificateAuthority` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `IssuePrecertificate` | `IssueCertificateRequest { bytes csr, int64 registrationID, int64 orderID, int64 issuerNameID, string certProfileName }` | `IssuePrecertificateResponse { bytes DER, bytes certProfileHash }` | Issue a precertificate (TBS) |
| `IssueCertificateForPrecertificate` | `IssueCertificateForPrecertificateRequest { bytes DER, repeated bytes SCTs, int64 registrationID, int64 orderID, bytes certProfileHash }` | `core.Certificate` | Issue a final certificate from a precertificate + SCTs |

---

#### Service: `OCSPGenerator` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `GenerateOCSP` | `GenerateOCSPRequest { string status, int32 reason, Timestamp revokedAt, string serial, int64 issuerID }` | `OCSPResponse { bytes response }` | Generate a signed OCSP response |

---

#### Service: `CRLGenerator` (`ca/proto/ca.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `GenerateCRL` | `stream GenerateCRLRequest` (oneof: `CRLMetadata { int64 issuerNameID, Timestamp thisUpdate, int64 shardIdx }` or `core.CRLEntry`) | `stream GenerateCRLResponse { bytes chunk }` | Generate a CRL via bidirectional streaming |

---

#### Service: `CRLStorer` (`crl/storer/proto/storer.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `UploadCRL` | `stream UploadCRLRequest` (oneof: `CRLMetadata { int64 issuerNameID, int64 number, int64 shardIdx }` or `bytes crlChunk`) | `google.protobuf.Empty` | Upload a CRL to storage (e.g. S3) via client-streaming |

---

#### Service: `Chiller` (`grpc/test_proto/interceptors_test.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Chill` | `Time { google.protobuf.Duration duration }` | `Time { google.protobuf.Duration duration }` | Test interceptor: sleep for the given duration |

---

#### Service: `NonceService` (`nonce/proto/nonce.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `Nonce` | `google.protobuf.Empty` | `NonceMessage { string nonce }` | Generate a fresh anti-replay nonce |
| `Redeem` | `NonceMessage { string nonce }` | `ValidMessage { bool valid }` | Redeem (validate + consume) a nonce |

---

#### Service: `Publisher` (`publisher/proto/publisher.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `SubmitToSingleCTWithResult` | `Request { bytes der, string LogURL, string LogPublicKey, SubmissionType kind }` | `Result { bytes sct }` | Submit a precertificate or certificate to a single CT log and return the SCT |

`SubmissionType` enum values: `unknown(0)`, `sct(1)`, `info(2)`, `final(3)`

---

#### Service: `RegistrationAuthority` (`ra/proto/ra.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `NewRegistration` | `core.Registration` | `core.Registration` | Create a new ACME account registration |
| `UpdateRegistration` | `UpdateRegistrationRequest { core.Registration base, core.Registration update }` | `core.Registration` | Update an existing registration |
| `PerformValidation` | `PerformValidationRequest { core.Authorization authz, int64 challengeIndex }` | `core.Authorization` | Trigger challenge validation |
| `DeactivateRegistration` | `core.Registration` | `google.protobuf.Empty` | Deactivate an account |
| `DeactivateAuthorization` | `core.Authorization` | `google.protobuf.Empty` | Deactivate an authorization |
| `RevokeCertByApplicant` | `RevokeCertByApplicantRequest { bytes cert, int64 code, int64 regID }` | `google.protobuf.Empty` | Revoke a certificate by its owner |
| `RevokeCertByKey` | `RevokeCertByKeyRequest { bytes cert }` | `google.protobuf.Empty` | Revoke a certificate by key compromise |
| `AdministrativelyRevokeCertificate` | `AdministrativelyRevokeCertificateRequest { bytes cert, string serial, int64 code, string adminName, bool skipBlockKey, bool malformed }` | `google.protobuf.Empty` | Administrative revocation |
| `NewOrder` | `NewOrderRequest { int64 registrationID, repeated string names, string replacesSerial, bool limitsExempt, string certificateProfileName }` | `core.Order` | Create a new certificate order |
| `FinalizeOrder` | `FinalizeOrderRequest { core.Order order, bytes csr }` | `core.Order` | Finalize an order with a CSR |
| `GenerateOCSP` | `GenerateOCSPRequest { string serial }` | `ca.OCSPResponse` | Generate OCSP response for a serial (RA-level) |

---

#### Service: `StorageAuthorityReadOnly` (`sa/proto/sa.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `CountCertificatesByNames` | `CountCertificatesByNamesRequest` | `CountByNames` | Count certificates issued per name |
| `CountFQDNSets` | `CountFQDNSetsRequest` | `Count` | Count FQDN sets |
| `CountInvalidAuthorizations2` | `CountInvalidAuthorizationsRequest` | `Count` | Count invalid authorizations |
| `CountOrders` | `CountOrdersRequest` | `Count` | Count orders |
| `CountPendingAuthorizations2` | `RegistrationID` | `Count` | Count pending authorizations for a registration |
| `CountRegistrationsByIP` | `CountRegistrationsByIPRequest` | `Count` | Count registrations by exact IP |
| `CountRegistrationsByIPRange` | `CountRegistrationsByIPRequest` | `Count` | Count registrations by IP range |
| `FQDNSetExists` | `FQDNSetExistsRequest` | `Exists` | Check if an FQDN set exists |
| `FQDNSetTimestampsForWindow` | `CountFQDNSetsRequest` | `Timestamps` | Get timestamps of FQDN sets within window |
| `GetAuthorization2` | `AuthorizationID2` | `core.Authorization` | Fetch authorization by ID |
| `GetAuthorizations2` | `GetAuthorizationsRequest` | `Authorizations` | Fetch multiple authorizations |
| `GetCertificate` | `Serial` | `core.Certificate` | Fetch a certificate by serial |
| `GetLintPrecertificate` | `Serial` | `core.Certificate` | Fetch a lint precertificate by serial |
| `GetCertificateStatus` | `Serial` | `core.CertificateStatus` | Fetch status of a certificate |
| `GetMaxExpiration` | `google.protobuf.Empty` | `google.protobuf.Timestamp` | Get maximum certificate expiration in DB |
| `GetOrder` | `OrderRequest` | `core.Order` | Fetch an order by ID |
| `GetOrderForNames` | `GetOrderForNamesRequest` | `core.Order` | Fetch an order by name set |
| `GetPendingAuthorization2` | `GetPendingAuthorizationRequest` | `core.Authorization` | Fetch a pending authorization |
| `GetRegistration` | `RegistrationID` | `core.Registration` | Fetch registration by ID |
| `GetRegistrationByKey` | `JSONWebKey` | `core.Registration` | Fetch registration by JWK |
| `GetRevocationStatus` | `Serial` | `RevocationStatus` | Get revocation status for a serial |
| `GetRevokedCerts` | `GetRevokedCertsRequest` | `stream core.CRLEntry` | Stream revoked certificate entries |
| `GetSerialMetadata` | `Serial` | `SerialMetadata` | Get metadata for a serial number |
| `GetSerialsByAccount` | `RegistrationID` | `stream Serial` | Stream serials for an account |
| `GetSerialsByKey` | `SPKIHash` | `stream Serial` | Stream serials for a public key |
| `GetValidAuthorizations2` | `GetValidAuthorizationsRequest` | `Authorizations` | Fetch valid authorizations |
| `GetValidOrderAuthorizations2` | `GetValidOrderAuthorizationsRequest` | `Authorizations` | Fetch valid authorizations for an order |
| `IncidentsForSerial` | `Serial` | `Incidents` | List incidents affecting a serial |
| `KeyBlocked` | `SPKIHash` | `Exists` | Check whether a key is blocked |
| `PreviousCertificateExists` | `PreviousCertificateExistsRequest` | `Exists` | Check for a prior certificate |
| `ReplacementOrderExists` | `Serial` | `Exists` | Check for a replacement order |
| `SerialsForIncident` | `SerialsForIncidentRequest` | `stream IncidentSerial` | Stream serials associated with an incident |

---

#### Service: `StorageAuthority` (`sa/proto/sa.proto`)

Exposes all `StorageAuthorityReadOnly` RPCs (identical signatures) plus the following write RPCs:

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `AddBlockedKey` | `AddBlockedKeyRequest` | `google.protobuf.Empty` | Add a key to the blocked-keys list |
| `AddCertificate` | `AddCertificateRequest` | `google.protobuf.Empty` | Persist a final certificate |
| `AddPrecertificate` | `AddCertificateRequest` | `google.protobuf.Empty` | Persist a precertificate |
| `SetCertificateStatusReady` | `Serial` | `google.protobuf.Empty` | Mark a certificate's status as ready |
| `AddSerial` | `AddSerialRequest` | `google.protobuf.Empty` | Record a serial number |
| `DeactivateAuthorization2` | `AuthorizationID2` | `google.protobuf.Empty` | Deactivate an authorization in the SA |
| `DeactivateRegistration` | `RegistrationID` | `google.protobuf.Empty` | Deactivate a registration in the SA |
| `FinalizeAuthorization2` | `FinalizeAuthorizationRequest` | `google.protobuf.Empty` | Finalize an authorization |
| `FinalizeOrder` | `FinalizeOrderRequest` | `google.protobuf.Empty` | Finalize an order in the SA |
| `NewOrderAndAuthzs` | `NewOrderAndAuthzsRequest` | `core.Order` | Atomically create an order and its authorizations |
| `NewRegistration` | `core.Registration` | `core.Registration` | Create a registration in the SA |
| `RevokeCertificate` | _Not determinable from code (truncated proto)_ | `google.protobuf.Empty` | Revoke a certificate in the SA |
| `SetOrderError` | _Not determinable from code (truncated proto)_ | `google.protobuf.Empty` | Record an error on an order |
| `SetOrderProcessing` | _Not determinable from code (truncated proto)_ | `google.protobuf.Empty` | Mark an order as being processed |
| `UpdateRevokedCertificate` | _Not determinable from code (truncated proto)_ | `google.protobuf.Empty` | Update revocation info for a certificate |
| `LeaseCRLShard` | _Not determinable from code (truncated proto)_ | _Not determinable from code._ | Lease a CRL shard for generation |
| `UpdateCRLShard` | _Not determinable from code (truncated proto)_ | `google.protobuf.Empty` | Update CRL shard metadata |

---

#### Service: `VA` (`va/proto/va.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `PerformValidation` | `PerformValidationRequest { string domain, core.Challenge challenge, AuthzMeta authz }` | `ValidationResult { repeated core.ValidationRecord records, core.ProblemDetails problems }` | Perform ACME challenge validation (HTTP-01, DNS-01, TLS-ALPN-01) |

#### Service: `CAA` (`va/proto/va.proto`)

| RPC | Request | Response | Purpose |
|-----|---------|----------|---------|
| `IsCAAValid` | `IsCAAValidRequest { string domain, string validationMethod, int64 accountURIID }` | `IsCAAValidResponse { core.ProblemDetails problem }` | Validate CAA DNS records for a domain |

---

### Core shared message types (`core/proto/core.proto`)

| Message | Key Fields |
|---------|-----------|
| `Certificate` | `int64 registrationID`, `string serial`, `string digest`, `bytes der`, `Timestamp issued`, `Timestamp expires` |
| `CertificateStatus` | `string serial`, `string status`, `Timestamp ocspLastUpdated`, `Timestamp revokedDate`, `int64 revokedReason`, `bool isExpired`, `int64 issuerID` |
| `Registration` | `int64 id`, `bytes key`, `repeated string contact`, `bool contactsPresent`, `string agreement`, `bytes initialIP`, `Timestamp createdAt`, `string status` |
| `Authorization` | `string id`, `string identifier`, `int64 registrationID`, `string status`, `Timestamp expires`, `repeated Challenge challenges` |
| `Order` | `int64 id`, `int64 registrationID`, `Timestamp expires`, `ProblemDetails error`, `string certificateSerial`, `string status`, `repeated string names`, `bool beganProcessing`, `Timestamp created`, `repeated int64 v2Authorizations`, `string certificateProfileName` |
| `Challenge` | `int64 id`, `string type`, `string status`, `string token`, `string keyAuthorization`, `repeated ValidationRecord validationrecords`, `ProblemDetails error`, `Timestamp validated` |
| `ProblemDetails` | `string problemType`, `string detail`, `int32 httpStatus` |
| `CRLEntry` | `string serial`, `int32 reason`, `Timestamp revokedAt` |

---

## Event Schemas

_Not determinable from code._

No message queue, Kafka topics, RabbitMQ exchanges, or other async event bus integrations are present in the extracted facts or source artifacts. All inter-service communication is synchronous gRPC.

---

## Input / Output Formats

**Serialization:** Protocol Buffers (proto3) exclusively for all gRPC service interfaces. Wire format is the standard protobuf binary encoding over HTTP/2 (gRPC framing).

**Timestamps:** All timestamps use `google.protobuf.Timestamp` (nanosecond precision UTC). Legacy nanosecond integer fields have been reserved/removed in favor of this type.

**Streaming patterns:**
- **Server-streaming:** `GetRevokedCerts` → `stream core.CRLEntry`; `GetSerialsByAccount` → `stream Serial`; `GetSerialsByKey` → `stream Serial`; `SerialsForIncident` → `stream IncidentSerial`; `GenerateCRL` response side.
- **Client-streaming:** `UploadCRL` request side.
- **Bidirectional streaming:** `GenerateCRL` (client streams `GenerateCRLRequest` containing either `CRLMetadata` or `CRLEntry` entries; server streams back `GenerateCRLResponse` chunks).

**Binary fields:** DER-encoded certificates and CSRs are transmitted as `bytes`. Public key hashes (`SPKIHash`) are `binary(32)`. SCTs are raw `bytes`.

**Pagination:** _Not determinable from code._ Streaming RPCs are used in place of pagination for large result sets (e.g., `GetSerialsByAccount`, `GetRevokedCerts`).

**Content-type negotiation:** Standard gRPC content type (`application/grpc+proto`). No REST content-type negotiation is present in the internal API layer.

**Observability:** OpenTelemetry tracing (`otelgrpc`) and Prometheus metrics (`go-grpc-prometheus`) are applied as gRPC interceptors, injecting trace context and metrics instrumentation into all gRPC calls.

---

## Error Handling

**gRPC status codes:** Boulder uses standard gRPC status codes propagated through the `google.golang.org/grpc/status` and `google.golang.org/genproto/googleapis/rpc` packages. Specific code-to-status mappings beyond what is implied by the proto definitions are _Not determinable from code_ from the available snapshot.

**`ProblemDetails` message:** Application-level errors (ACME problem documents) are carried in-band as `core.ProblemDetails { string problemType, string detail, int32 httpStatus }`. This message is embedded in:
- `core.Challenge.error`
- `core.Order.error`
- `va.IsCAAValidResponse.problem` — an empty `problem` field signals CAA is valid.
- `va.ValidationResult.problems`

**Validation failure signaling:** For CAA validation (`IsCAAValid`), a populated `ProblemDetails` in the response indicates failure; an empty/absent problem indicates success. For challenge validation (`PerformValidation`), failure is indicated by a populated `problems` field in `ValidationResult`.

**Administrative revocation:** The `malformed` flag in `AdministrativelyRevokeCertificateRequest` prevents certificate parsing; when set, `keyCompromise` reason code cannot be used (key blocking is not possible without parsing).

**Redis usage:** Redis (`go-redis/v9`) is used for OCSP response caching and rate-limit storage; error handling for cache misses is _Not determinable from code_ from the available artifacts.

**Database errors:** MySQL accessed via `go-sql-driver/mysql` through the `borp` ORM. Versions >1.5.0 are explicitly excluded in `go.mod` due to documented performance regressions.

---

## Versioning

**Proto field reservation:** Boulder uses proto3 field reservation (`reserved`) extensively to tombstone deprecated fields (e.g., legacy nanosecond timestamp fields replaced by `google.protobuf.Timestamp`). This is the primary schema evolution mechanism — old field numbers are never reused.

**No URI versioning:** There are no HTTP URI version prefixes in the internal gRPC API surface.

**SA read/write split versioning:** The `StorageAuthorityReadOnly` service is a strict subset of `StorageAuthority` (identical read RPC signatures), enabling separate deployment and access-control tiers without API version divergence.

**Go module version:** `github.com/letsencrypt/boulder` at Go 1.21, with `google.golang.org/protobuf v1.33.0` and `google.golang.org/grpc v1.60.1`.

**No explicit API version header or URI versioning strategy** is present; schema evolution is managed through proto field reservation and additive changes only.
