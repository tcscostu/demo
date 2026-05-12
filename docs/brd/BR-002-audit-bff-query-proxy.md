# BR-002: Audit BFF Query Proxy

## 1. Summary
Deliver a lightweight Backend-for-Frontend (BFF) service that proxies audit event queries from the enterprise frontend application to the upstream Audit Event API. The BFF exposes a single `POST /audit/query` endpoint, accepts structured query parameters, forwards them to the upstream audit-event service, and returns the raw response to the frontend without transformation or business logic.

**User Story**: As an enterprise operations user, I want to query audit events through a dedicated BFF endpoint so that the frontend application can retrieve audit data without directly coupling to the internal audit platform API.

## 2. Business Rules
- **BR-002-01**: The BFF SHALL expose `POST /audit/query` accepting a JSON request body with filter fields: `eventName` (optional), `startDate` (required, dd/MM/yyyy HH:mm), `endDate` (required, dd/MM/yyyy HH:mm), `actorId` (optional), `status` (optional, enum: SUCCESS/FAILURE).
- **BR-002-02**: The BFF SHALL forward the validated request to the upstream Audit Event API (`/api/v1/audit-events`) preserving the same request and response structure.
- **BR-002-03**: The upstream audit platform base URL and service name MUST be configurable via `application.yml` properties. Actual values will be set at deployment time.
- **BR-002-04**: The BFF SHALL NOT implement any business logic. It acts purely as a proxy/pass-through layer between the frontend and the audit platform.
- **BR-002-05**: The BFF SHALL NOT connect to any database. No persistence layer is required.
- **BR-002-06**: Authentication SHALL NOT be implemented in the BFF. An inhouse enterprise secure API gateway handles authentication upstream.
- **BR-002-07**: The BFF SHALL return HTTP 200 with a raw JSON array of audit event objects on success. No response envelope or wrapper is used.
- **BR-002-08**: The BFF SHALL return RFC 9457 Problem Details format for all error responses (HTTP 400, 404, 500, 502, etc.).
- **BR-002-09**: The BFF SHALL expose Swagger UI at the root path (`/`) and OpenAPI spec at `/v3/api-docs` for API documentation.
- **BR-002-10**: No pagination is required in the BFF. Pagination is handled by the frontend application.
- **BR-002-11**: No resilience configuration (timeout, retry, circuit-breaker) is required. This BFF serves a small number of enterprise internal users only.

## 3. Acceptance Criteria / Scenarios
```gherkin
Scenario: Successful audit query with all filters
  Given the upstream audit platform is available
  When a caller POSTs to /audit/query with eventName, startDate, endDate, actorId, and status
  Then the BFF forwards the request to the upstream audit platform
  And returns HTTP 200 with a JSON array of audit event objects

Scenario: Successful audit query with required filters only
  Given the upstream audit platform is available
  When a caller POSTs to /audit/query with only startDate and endDate
  Then the BFF forwards the request to the upstream audit platform
  And returns HTTP 200 with a JSON array of matching audit events

Scenario: Missing required startDate field
  Given a request body without the startDate field
  When a caller POSTs to /audit/query
  Then the BFF returns HTTP 400 Bad Request with RFC 9457 Problem Details
  And the detail field describes the missing startDate

Scenario: Missing required endDate field
  Given a request body without the endDate field
  When a caller POSTs to /audit/query
  Then the BFF returns HTTP 400 Bad Request with RFC 9457 Problem Details
  And the detail field describes the missing endDate

Scenario: Invalid status value
  Given a request body with status "UNKNOWN"
  When a caller POSTs to /audit/query
  Then the BFF returns HTTP 400 Bad Request with RFC 9457 Problem Details

Scenario: Upstream audit platform is unavailable
  Given the upstream audit platform is not reachable
  When a caller POSTs to /audit/query with valid parameters
  Then the BFF returns HTTP 502 Bad Gateway with RFC 9457 Problem Details

Scenario: Upstream audit platform returns an error
  Given the upstream audit platform returns HTTP 500
  When a caller POSTs to /audit/query with valid parameters
  Then the BFF propagates the error as an appropriate HTTP status with RFC 9457 Problem Details

Scenario: Swagger UI is accessible
  Given the BFF application is running
  When a user navigates to /
  Then the Swagger UI is displayed with the /audit/query endpoint documented

Scenario: Invalid date format
  Given a request body with startDate "2026-02-01 00:00" (wrong format)
  When a caller POSTs to /audit/query
  Then the BFF returns HTTP 400 Bad Request with RFC 9457 Problem Details
  And the detail field describes the invalid date format
```

## 4. REST API Endpoint

### 4.1 Query Audit Events
```
POST /audit/query
Content-Type: application/json

Request Body:
{
  "eventName": "FAST_GONDERIM",         // string, optional
  "startDate": "01/02/2026 00:00",       // string, dd/MM/yyyy HH:mm, required
  "endDate": "19/02/2026 23:59",         // string, dd/MM/yyyy HH:mm, required
  "actorId": "CUST-20481001",            // string, optional
  "status": "SUCCESS"                    // enum: SUCCESS/FAILURE, optional
}

Response: 200 OK
Content-Type: application/json

[
  {
    "eventName": "FAST_GONDERIM",
    "outcome": {
      "status": "SUCCESS",
      "statusCode": "1000",
      "description": "EFT işleminiz başarıyla gerçekleştirilmiştir."
    },
    "sourceInfo": {
      "sourceSystem": "URUN",
      "sourceVersion": "3.8.1",
      "sourceClientVersion": "5.12.0",
      "sourceServiceName": "Complete",
      "devicePlatform": "IOS 27.1",
      "sourceDeviceIp": "85.103.42.117",
      "sourceIp": "10.20.30.44"
    },
    "actor": {
      "type": "CUSTOMER",
      "id": "CUST-20481001",
      "sessionId": "sess_f1a2b3c4-d5e6-7890-abcd-ef1234567890"
    },
    "customerInfo": {
      "crmCustomerId": "CUST-20481001",
      "crmAccountId": "ACC-20481001-001",
      "tckn": "123456****",
      "msisdn": "5321234****",
      "iban": "TR12-****-****-****-****-0048",
      "pmi": null,
      "token": null
    },
    "amountInfo": {
      "amount": 1000.00,
      "transactionAmount": 1000.00,
      "feeAmount": 2.50,
      "comissionAmount": 0.00,
      "bsmv": 0.13,
      "totalAmount": 1002.63,
      "refundAmount": null
    },
    "businessContext": {
      "filterKey": "TRF-2026-02-19-0047291",
      "eventDetail": {
        "transferType": "EFT",
        "currency": "TRY",
        "senderName": "Serkan Co",
        "senderIban": "TR12-****-****-****-****-0048",
        "receiverName": "Gamze Ko",
        "receiverIban": "TR67-****-****-****-****-3921",
        "receiverBank": "Garanti BBVA",
        "description": "Kira ödemesi"
      }
    },
    "diff": [
      {
        "property": "senderBalance",
        "oldValue": "12500.00",
        "newValue": "11497.37"
      }
    ],
    "correlation": {
      "traceId": "999-9999-9999-aaa111bbb222",
      "correlationId": "corr_a1b2c3d4-e5f6-7890-abcd-111111111111",
      "referenceId": "TRF-2026-02-19-0047291",
      "sourceProcessDate": "2026-02-19T14:23:07.441Z"
    }
  }
]
```

### 4.2 Error Response Format (RFC 9457)
```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR",
  "title": "Validation Error",
  "status": 400,
  "detail": "startDate is required and must be in dd/MM/yyyy HH:mm format",
  "instance": "/audit/query",
  "timestamp": "2026-03-09T10:30:00.123Z"
}
```

### 4.3 Upstream Proxy Flow
```
Frontend → [POST /audit/query] → Audit BFF → [upstream call] → Audit Event API
                                    ↓
                            Validate request
                            Forward to upstream
                            Return raw response
```

## 5. Technology Stack
| Component | Technology | Version | Rationale |
|-----------|------------|---------|-----------|
| Runtime | Java (OpenJDK) | 21 LTS | Matches enterprise BFF standard (bo-bff-fsm) |
| Language | Kotlin | 1.9.25 | Matches enterprise BFF standard (bo-bff-fsm) |
| Framework | Spring Boot | 3.5.3 | Matches enterprise BFF standard (bo-bff-fsm) |
| Build Tool | Maven | 3.9.x | Matches enterprise BFF standard (bo-bff-fsm) |
| Server | Embedded Tomcat | (via Spring Boot) | Default servlet container |
| API Docs | SpringDoc OpenAPI | 2.6.0 | Swagger UI generation |
| HTTP Client | Spring RestClient | (via Spring Boot) | Upstream proxy calls |
| Logging | SLF4J + Spring Boot | (via Spring Boot) | Structured logging with AOP |
| Code Quality | Spotless (ktlint) | 2.43.0 (1.0.1) | Code formatting |
| Test Coverage | JaCoCo | 0.8.11 | Coverage reporting |
| Testing | JUnit 5 + MockK | 5.x / 1.13.8 | Unit and integration testing |
| Tracing | OpenTelemetry API | 1.31.0+ | Distributed tracing support |

## 6. Cross-Cutting Concerns Checklist (Planning Pact)
| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|------------------|--------------|----------|
| Runtime & Framework | ADR-01 | Java 21, Spring Boot 3.5.3, Kotlin 1.9.25, Maven | TBD |
| API Documentation | ADR-02 | SpringDoc OpenAPI 2.6.0, Swagger UI at `/` | TBD |
| Package Structure | ADR-03 | Simplified hexagonal: adapter, domain, config | TBD |
| Application Configuration | ADR-04 | `application.yml` with configurable upstream URL | TBD |
| Error Handling | ADR-05 | RFC 9457 Problem Details via `@RestControllerAdvice` | TBD |
| Structured Logging | ADR-06 | AOP-based REST request/response logging | TBD |
| Code Quality | ADR-07 | Spotless (ktlint), JaCoCo coverage | TBD |
| Upstream Proxy | ADR-08 | RestClient forwarding to upstream Audit Event API | TBD |
| Data Transfer Validation | ADR-09 | Jakarta Bean Validation on request DTO | TBD |
| Testing Strategy | ADR-10 | JUnit 5 + MockK, MockMvc controller tests | TBD |
| Database | N/A | Explicitly out of scope — no database dependency | N/A |
| Authentication | N/A | Handled by enterprise API gateway — not implemented | N/A |
| Pagination | N/A | Handled by frontend — not implemented in BFF | N/A |
| Resilience | N/A | Not required — internal enterprise use only | N/A |

## 7. Requirements Coverage Matrix
| Requirement ID | Description | Planned Artifact | Status |
|----------------|-------------|------------------|--------|
| BR-002-01 | POST /audit/query endpoint | `AuditQueryController.kt` | TBD |
| BR-002-02 | Forward to upstream Audit Event API | `AuditPlatformRestAdapter.kt` | TBD |
| BR-002-03 | Configurable upstream URL | `application.yml` property | TBD |
| BR-002-04 | No business logic | Proxy-only controller | TBD |
| BR-002-05 | No database | No JPA/datasource dependencies | TBD |
| BR-002-06 | No authentication | No security dependencies | TBD |
| BR-002-07 | Raw JSON array response | Controller returns `List<AuditEvent>` | TBD |
| BR-002-08 | RFC 9457 error responses | `GlobalExceptionHandler.kt` | TBD |
| BR-002-09 | Swagger UI | SpringDoc OpenAPI config | TBD |
| BR-002-10 | No pagination | Not implemented | TBD |
| BR-002-11 | No resilience config | No circuit-breaker/retry dependencies | TBD |

## 8. Related ADRs
- ADR-01: Runtime and Framework (Java 21, Spring Boot 3.5.3, Kotlin 1.9.25)
- ADR-02: API Documentation (SpringDoc OpenAPI)
- ADR-03: Package Structure Organization
- ADR-04: Application Configuration Management
- ADR-05: REST API Error Handling (RFC 9457)
- ADR-06: Structured Logging Strategy
- ADR-07: Code Quality Standards
- ADR-08: Upstream Proxy Integration Pattern
- ADR-09: Data Transfer Validation
- ADR-10: Testing Strategy (JUnit 5, MockK, MockMvc)

## 9. Risks & Mitigations
| Risk | Impact | Mitigation |
|------|--------|-----------|
| Upstream API unavailable | BFF returns errors to frontend | Return clear 502 error with Problem Details; upstream URL is configurable for failover |
| Upstream API contract changes | BFF may forward/receive unexpected data | Since BFF is pass-through, contract changes flow through; monitor upstream versioning |
| No resilience patterns | Requests may hang if upstream is slow | Acceptable for internal enterprise use with small user base; add timeout config if needed later |
| No authentication in BFF | Unauthorized access possible if gateway misconfigured | API gateway provides security boundary; BFF operates within secure internal network |

## 10. Open Questions / Future Enhancements
| Item | Decision Needed By | Notes |
|------|--------------------|-------|
| Upstream base URL | Before deployment | Configurable via `application.yml`, actual value TBD |
| Response caching | Post-MVP | May add short-lived cache for repeated queries |
| Rate limiting | If exposed beyond internal use | Not needed for current small enterprise user base |
| Pagination support | If query result sets grow large | Currently handled by frontend |

## 11. Approval
| Role | Name | Date | Status |
|------|------|------|--------|
| Tech Lead | TBD | TBD | Pending |
| Architect | TBD | TBD | Pending |
| Product Owner | TBD | TBD | Pending |

---
End of BR-002.
