# PROJECT OVERVIEW — Audit BFF Query Proxy

## Summary

The Audit BFF (Backend-for-Frontend) is a lightweight proxy service that exposes a single POST endpoint (`/audit/query`) for the frontend to query audit events. It validates the request, forwards it to the upstream Audit Event API, and returns the response as a raw JSON array. The service has no database, no authentication (handled by API gateway), and no pagination (handled by frontend).

## Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Kotlin | 1.9.25 |
| Runtime | Java (OpenJDK) | 21 LTS |
| Framework | Spring Boot | 3.5.3 |
| Build Tool | Maven | 3.9.x |
| API Docs | SpringDoc OpenAPI | 2.6.0 |
| Validation | Jakarta Bean Validation | 3.0.x |
| HTTP Client | RestClient | (Spring Boot bundled) |
| Code Format | Spotless + ktlint | 2.43.0 / 1.0.1 |
| Coverage | JaCoCo | 0.8.11 |
| Testing | JUnit 5 + MockK | 5.x / 1.13.8 |

## API Endpoint

| Method | Path | Description |
|--------|------|-------------|
| POST | `/audit/query` | Query audit events from upstream platform |

## Request Fields

| Field | Type | Required | Constraint |
|-------|------|----------|------------|
| `eventName` | String | No | — |
| `startDate` | String | Yes | `dd/MM/yyyy HH:mm` |
| `endDate` | String | Yes | `dd/MM/yyyy HH:mm` |
| `actorId` | String | No | — |
| `status` | Enum | No | `SUCCESS` \| `FAILURE` |

## Key Architectural Decisions

- **Proxy-only**: No business logic, no data transformation, no database
- **Upstream**: Configurable via `audit.platform.base-url` property
- **Error format**: RFC 9457 Problem Details
- **Response**: Raw JSON array from upstream (no wrapper/envelope)
- **Logging**: AOP-based REST request/response logging

## Package Structure

```
tr.com.mycorp.audit.bff
├── adapter
│   ├── inbound          # REST controllers
│   └── outbound         # Upstream HTTP client (RestClient)
├── config               # Spring configuration, properties
├── domain
│   └── model
│       ├── request      # AuditQueryRequest, AuditStatus
│       └── response     # AuditEvent and nested DTOs
├── aspect               # AOP logging
└── exception            # GlobalExceptionHandler, UpstreamServiceException
```

## Architecture Decision Records

| # | ADR | Scope |
|---|-----|-------|
| 00 | [ADR Structure Summary](00-adr-structure-summary.md) | Template and index |
| 01 | [Runtime and Framework](01-runtime-and-framework.md) | Java 21 + Spring Boot 3.5.3 + Kotlin + Maven |
| 02 | [API Documentation](02-api-documentation.md) | SpringDoc OpenAPI / Swagger UI |
| 03 | [Package Structure](03-package-structure.md) | Simplified hexagonal layout |
| 04 | [Application Configuration](04-application-configuration.md) | application.yml + upstream properties |
| 05 | [REST API Error Handling](05-rest-api-error-handling.md) | RFC 9457 Problem Details |
| 06 | [Structured Logging](06-structured-logging.md) | AOP-based request/response logging |
| 07 | [Code Quality](07-code-quality.md) | Spotless + ktlint + JaCoCo |
| 08 | [Upstream Proxy Integration](08-upstream-proxy-integration.md) | RestClient proxy to upstream Audit Event API |
| 09 | [Data Transfer Validation](09-data-transfer-validation.md) | Jakarta Bean Validation on DTOs |
| 10 | [Testing Strategy](10-testing-strategy.md) | JUnit 5 + MockK + MockMvc |

## Business Requirements

| # | BRD | Description |
|---|-----|-------------|
| BR-002 | [Audit BFF Query Proxy](../brd/BR-002-audit-bff-query-proxy.md) | BFF proxy service for audit event querying |

## Upstream Dependency

- **Audit Event API**: The BFF proxies requests to this service at `POST /api/v1/audit-events`
- Base URL is configured via `audit.platform.base-url` property (environment variable: `AUDIT_PLATFORM_BASE_URL`)

## What This Service Does NOT Do

- ❌ Database access (no JPA, no datasource)
- ❌ Authentication/authorization (API gateway responsibility)
- ❌ Pagination (frontend responsibility)
- ❌ Response transformation or aggregation
- ❌ Resilience patterns (internal enterprise use)
- ❌ Asynchronous processing
