# Issue: Package Structure Organization for Audit BFF

The Audit BFF requires a clear, maintainable package structure that follows the enterprise BFF conventions while adapting for a simple proxy service with no database or complex domain logic.

**Impact**: Without standardized package organization, code becomes disorganized, new developers cannot locate components, and architectural boundaries erode over time.

# Decision:

Implement a simplified package structure inspired by the enterprise BFF pattern (bo-bff-fsm) and hexagonal architecture principles, adapted for a stateless proxy BFF with no persistence layer.

**Required Structure**:
```
tr.com.mycorp.audit.bff/
├── BffAuditApplication.kt              # Spring Boot entry point
├── adapter/
│   ├── inbound/
│   │   └── AuditQueryController.kt     # REST controller (inbound adapter)
│   └── outbound/
│       └── AuditPlatformRestAdapter.kt # HTTP client to upstream API (outbound adapter)
├── domain/
│   └── model/
│       ├── request/
│       │   └── AuditQueryRequest.kt    # Inbound request DTO
│       └── response/
│           └── AuditEvent.kt           # Response model (mirrors upstream)
├── config/
│   ├── OpenApiConfig.kt                # Swagger/OpenAPI configuration
│   ├── RestClientConfig.kt             # RestClient bean configuration
│   ├── AopConfiguration.kt             # Enable AOP auto-proxy
│   ├── CorsProperties.kt              # CORS configuration properties
│   └── WebConfig.kt                    # WebMvc CORS registry
├── aspect/
│   └── RestControllerLoggingAspect.kt  # AOP request/response logging
└── exception/
    ├── GlobalExceptionHandler.kt       # @RestControllerAdvice RFC 9457
    ├── UpstreamServiceException.kt     # Upstream communication errors
    └── ValidationException.kt          # Request validation errors
```

# Constraints:

- Package root MUST be `tr.com.mycorp.audit.bff` following enterprise naming convention
- Controller (inbound adapter) MUST reside in `adapter.inbound` package
- Upstream HTTP client (outbound adapter) MUST reside in `adapter.outbound` package
- Request/response models MUST reside in `domain.model.request` and `domain.model.response`
- No `repository`, `service`, or `entity` packages — no database or business logic
- Configuration classes MUST reside in `config` package
- Exception/error handling MUST reside in `exception` package
- AOP aspects MUST reside in `aspect` package

# Alternatives:

**Flat package structure**: All classes in root package. Rejected because it doesn't scale and violates enterprise conventions.

**Full hexagonal with ports**: Complete port/adapter separation with interfaces. Rejected as over-engineering for a single-endpoint proxy BFF with no business logic to abstract.

**Feature-based packaging**: Organize by feature (e.g., `audit.query`). Rejected because the BFF has only one feature — a query proxy — making feature-based packaging unnecessary.

# Rationale:

The structure preserves the key hexagonal principle of separating inbound adapters (REST controller) from outbound adapters (HTTP client) while eliminating layers that add no value for a simple proxy (service layer, repository, port interfaces). The `domain.model` package holds DTOs that represent the API contract. This matches the spirit of bo-bff-fsm's organization (`adapter/`, `domain/model/`, `config/`) without the JPA-specific packages.

# Implementation Guidelines:

## 1. Directory Creation

```bash
# Source directories
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/adapter/inbound
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/adapter/outbound
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/domain/model/request
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/domain/model/response
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/config
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/aspect
mkdir -p src/main/kotlin/tr/com/mycorp/audit/bff/exception

# Test directories (mirror source structure)
mkdir -p src/test/kotlin/tr/com/mycorp/audit/bff/adapter/inbound
mkdir -p src/test/kotlin/tr/com/mycorp/audit/bff/adapter/outbound
```

## 2. Component Placement Guide

| Component | Package | Class Name |
|-----------|---------|------------|
| Application entry | `tr.com.mycorp.audit.bff` | `BffAuditApplication.kt` |
| REST controller | `adapter.inbound` | `AuditQueryController.kt` |
| Upstream HTTP client | `adapter.outbound` | `AuditPlatformRestAdapter.kt` |
| Request DTO | `domain.model.request` | `AuditQueryRequest.kt` |
| Response model | `domain.model.response` | `AuditEvent.kt` (+ nested classes) |
| OpenAPI config | `config` | `OpenApiConfig.kt` |
| RestClient config | `config` | `RestClientConfig.kt` |
| AOP config | `config` | `AopConfiguration.kt` |
| CORS config | `config` | `CorsProperties.kt`, `WebConfig.kt` |
| Logging aspect | `aspect` | `RestControllerLoggingAspect.kt` |
| Exception handler | `exception` | `GlobalExceptionHandler.kt` |
| Custom exceptions | `exception` | `UpstreamServiceException.kt`, `ValidationException.kt` |

## 3. Naming Conventions

- **Controllers**: `[Feature]Controller.kt` (e.g., `AuditQueryController`)
- **Outbound Adapters**: `[Service]RestAdapter.kt` (e.g., `AuditPlatformRestAdapter`)
- **Request DTOs**: `[Feature]Request.kt` (e.g., `AuditQueryRequest`)
- **Response DTOs**: `[Entity].kt` (e.g., `AuditEvent`)
- **Config classes**: `[Feature]Config.kt` (e.g., `OpenApiConfig`, `RestClientConfig`)
- **Aspects**: `[Concern]Aspect.kt` (e.g., `RestControllerLoggingAspect`)
- **Exceptions**: `[Error]Exception.kt` (e.g., `UpstreamServiceException`)

# Additional Recommendations:

- If additional proxy endpoints are added in the future, consider introducing a `service` package as an intermediary layer
- Keep the response model classes in `domain.model.response` as simple Kotlin `data class` types mirroring the upstream JSON structure
- Place test classes in mirrored package structure under `src/test/kotlin`

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Package Root | `find . -path "*mycorp/audit/bff*" -type d` | Root package exists |
| Inbound Adapter Package | `find . -path "*adapter/inbound*" -type d` | Directory exists |
| Outbound Adapter Package | `find . -path "*adapter/outbound*" -type d` | Directory exists |
| Model Package | `find . -path "*domain/model*" -type d` | Directory exists |
| Config Package | `find . -path "*bff/config*" -type d` | Directory exists |
| Exception Package | `find . -path "*bff/exception*" -type d` | Directory exists |
| No Repository Package | `find . -path "*repository*" -type d \| wc -l` | `0` |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Package Structure..."

REQUIRED_PACKAGES=(
  "adapter/inbound"
  "adapter/outbound"
  "domain/model/request"
  "domain/model/response"
  "config"
  "aspect"
  "exception"
)

for pkg in "${REQUIRED_PACKAGES[@]}"; do
  if find . -path "*mycorp/audit/bff/$pkg" -type d | grep -q .; then
    echo "✅ Package '$pkg' exists"
  else
    echo "⚠️ Package '$pkg' not found"
  fi
done

# Verify no database-related packages
FORBIDDEN_PACKAGES=("repository" "entity" "persistence")
for pkg in "${FORBIDDEN_PACKAGES[@]}"; do
  if find . -path "*mycorp/audit/bff/*$pkg*" -type d | grep -q .; then
    echo "❌ Forbidden package '$pkg' found — no database in this BFF"
    exit 1
  else
    echo "✅ No forbidden package '$pkg'"
  fi
done

echo "✅ All Package Structure criteria met"
```
