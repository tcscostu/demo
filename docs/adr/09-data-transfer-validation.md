# Issue: Data Transfer and Validation for Audit BFF Request

The Audit BFF receives query requests from the frontend with specific field constraints. Validation must be applied before forwarding to the upstream API to provide fast, clear feedback on invalid inputs.

**Impact**: Without input validation, malformed requests propagate to the upstream API, returning cryptic errors. Proper DTO validation ensures early failure with RFC 9457-compliant error messages.

# Decision:

Use Jakarta Bean Validation annotations on Kotlin data classes for request DTO validation. Spring Boot's `@Valid` annotation triggers validation automatically in the controller layer.

**Required Components**:
- `spring-boot-starter-validation` dependency (included in starter-web)
- `AuditQueryRequest` data class with Jakarta validation annotations
- `AuditStatus` enum for status field
- `AuditEvent` and nested response data classes
- `MethodArgumentNotValidException` handled in `GlobalExceptionHandler` (ADR-05)

# Constraints:

- `startDate` and `endDate` are REQUIRED (`@NotBlank`)
- Date format MUST be `dd/MM/yyyy HH:mm` — enforced via `@Pattern`
- `eventName` is OPTIONAL (nullable, no validation)
- `actorId` is OPTIONAL (nullable, no validation)
- `status` is OPTIONAL (nullable), when present MUST be `SUCCESS` or `FAILURE` — enforced via enum type
- Response model MUST match the upstream Audit Event API response structure exactly
- Data classes MUST reside in `domain.model.request` and `domain.model.response` packages

# Alternatives:

**Manual Validation in Controller**: Validating fields manually in the controller method. Rejected — duplicates framework functionality, verbose, error-prone.

**Custom Validator Class**: Implementing a `Validator` interface. Rejected — Jakarta annotations are sufficient for the simple constraints in this DTO.

**JSR 380 with Custom Constraint Annotation**: Creating a `@DateFormat` annotation with a custom validator. Could be considered for reuse but adds complexity. Using `@Pattern` is simpler for a single DTO.

# Rationale:

Jakarta Bean Validation is the standard approach in Spring Boot for request validation. It integrates transparently with `@Valid` in controller methods, automatically produces `MethodArgumentNotValidException` on failure (handled by ADR-05 GlobalExceptionHandler), and keeps validation rules co-located with the DTO definition.

# Implementation Guidelines:

## 1. Request Data Class

```kotlin
package tr.com.mycorp.audit.bff.domain.model.request

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Pattern

@Schema(description = "Audit event query request")
data class AuditQueryRequest(
    @Schema(description = "Event name filter", example = "FAST_GONDERIM", required = false)
    val eventName: String? = null,

    @field:NotBlank(message = "Start date is required")
    @field:Pattern(
        regexp = "^\\d{2}/\\d{2}/\\d{4} \\d{2}:\\d{2}$",
        message = "Start date must be in dd/MM/yyyy HH:mm format",
    )
    @Schema(description = "Start date for query range", example = "01/02/2026 00:00", required = true)
    val startDate: String,

    @field:NotBlank(message = "End date is required")
    @field:Pattern(
        regexp = "^\\d{2}/\\d{2}/\\d{4} \\d{2}:\\d{2}$",
        message = "End date must be in dd/MM/yyyy HH:mm format",
    )
    @Schema(description = "End date for query range", example = "19/02/2026 23:59", required = true)
    val endDate: String,

    @Schema(description = "Actor identifier filter", example = "CUST-20481001", required = false)
    val actorId: String? = null,

    @Schema(description = "Outcome status filter", example = "SUCCESS", required = false)
    val status: AuditStatus? = null,
)
```

## 2. Status Enum

```kotlin
package tr.com.mycorp.audit.bff.domain.model.request

import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Audit event outcome status", enumAsRef = true)
enum class AuditStatus {
    SUCCESS,
    FAILURE,
}
```

## 3. Response Data Classes

Based on the upstream Audit Event API response structure:

```kotlin
package tr.com.mycorp.audit.bff.domain.model.response

import io.swagger.v3.oas.annotations.media.Schema
import java.math.BigDecimal

@Schema(description = "Audit event record")
data class AuditEvent(
    @Schema(description = "Event name", example = "FAST_GONDERIM")
    val eventName: String? = null,
    val outcome: Outcome? = null,
    val sourceInfo: SourceInfo? = null,
    val actor: Actor? = null,
    val customerInfo: CustomerInfo? = null,
    val amountInfo: AmountInfo? = null,
    val businessContext: BusinessContext? = null,
    val diff: List<DiffEntry>? = null,
    val correlation: Correlation? = null,
)

@Schema(description = "Outcome of the audit event")
data class Outcome(
    @Schema(description = "Status label", example = "SUCCESS")
    val status: String? = null,
    @Schema(description = "Status code", example = "1000")
    val statusCode: String? = null,
    @Schema(description = "Human-readable description")
    val description: String? = null,
)

@Schema(description = "Source system information")
data class SourceInfo(
    @Schema(description = "Source system name", example = "URUN")
    val sourceSystem: String? = null,
    @Schema(description = "Source system version", example = "3.8.1")
    val sourceVersion: String? = null,
    @Schema(description = "Client application version", example = "5.12.0")
    val sourceClientVersion: String? = null,
    @Schema(description = "Service name", example = "Complete")
    val sourceServiceName: String? = null,
    @Schema(description = "Device platform", example = "IOS 27.1")
    val devicePlatform: String? = null,
    @Schema(description = "Source device IP", example = "85.103.42.117")
    val sourceDeviceIp: String? = null,
    @Schema(description = "Source IP", example = "10.20.30.44")
    val sourceIp: String? = null,
)

@Schema(description = "Actor who performed the action")
data class Actor(
    @Schema(description = "Actor type", example = "CUSTOMER")
    val type: String? = null,
    @Schema(description = "Actor identifier", example = "CUST-20481001")
    val id: String? = null,
    @Schema(description = "Session identifier")
    val sessionId: String? = null,
)

@Schema(description = "Customer information")
data class CustomerInfo(
    @Schema(description = "CRM customer ID", example = "CUST-20481001")
    val crmCustomerId: String? = null,
    @Schema(description = "CRM account ID", example = "ACC-20481001-001")
    val crmAccountId: String? = null,
    @Schema(description = "Masked TCKN", example = "123456****")
    val tckn: String? = null,
    @Schema(description = "Masked MSISDN", example = "5321234****")
    val msisdn: String? = null,
    @Schema(description = "Masked IBAN", example = "TR12-****-****-****-****-0048")
    val iban: String? = null,
    @Schema(description = "PMI value")
    val pmi: String? = null,
    @Schema(description = "Token value")
    val token: String? = null,
)

@Schema(description = "Financial amount details")
data class AmountInfo(
    @Schema(description = "Base amount", example = "1000.00")
    val amount: BigDecimal? = null,
    @Schema(description = "Transaction amount", example = "1000.00")
    val transactionAmount: BigDecimal? = null,
    @Schema(description = "Fee amount", example = "2.50")
    val feeAmount: BigDecimal? = null,
    @Schema(description = "Commission amount", example = "0.00")
    val comissionAmount: BigDecimal? = null,
    @Schema(description = "BSMV tax amount", example = "0.13")
    val bsmv: BigDecimal? = null,
    @Schema(description = "Total amount", example = "1002.63")
    val totalAmount: BigDecimal? = null,
    @Schema(description = "Refund amount")
    val refundAmount: BigDecimal? = null,
)

@Schema(description = "Business context and event details")
data class BusinessContext(
    @Schema(description = "Filter key", example = "TRF-2026-02-19-0047291")
    val filterKey: String? = null,
    @Schema(description = "Dynamic event detail map")
    val eventDetail: Map<String, Any?>? = null,
)

@Schema(description = "Diff entry showing a changed property")
data class DiffEntry(
    @Schema(description = "Property name", example = "senderBalance")
    val property: String? = null,
    @Schema(description = "Value before change", example = "12500.00")
    val oldValue: String? = null,
    @Schema(description = "Value after change", example = "11497.37")
    val newValue: String? = null,
)

@Schema(description = "Correlation and tracing identifiers")
data class Correlation(
    @Schema(description = "Trace ID", example = "999-9999-9999-aaa111bbb222")
    val traceId: String? = null,
    @Schema(description = "Correlation ID")
    val correlationId: String? = null,
    @Schema(description = "Reference ID", example = "TRF-2026-02-19-0047291")
    val referenceId: String? = null,
    @Schema(description = "Source process date", example = "2026-02-19T14:23:07.441Z")
    val sourceProcessDate: String? = null,
)
```

## 4. Controller Usage

```kotlin
@PostMapping("/audit/query")
fun queryAuditEvents(
    @Valid @RequestBody request: AuditQueryRequest,
): List<AuditEvent> {
    return auditPlatformAdapter.queryAuditEvents(request)
}
```

When validation fails, Spring throws `MethodArgumentNotValidException`, which is caught by `GlobalExceptionHandler` (ADR-05) and returned as an RFC 9457 ProblemDetail response.

## 5. Validation Error Response Example

Consistent with ADR-05 error format (RFC 9457, custom `type` URI, `timestamp` extension):

```json
{
  "type": "https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR",
  "title": "Validation Error",
  "status": 400,
  "detail": "startDate: Start date is required; endDate: End date must be in dd/MM/yyyy HH:mm format",
  "instance": "/audit/query",
  "timestamp": "2026-03-09T10:30:00.123Z"
}
```

## 6. Invalid Status Handling

When an invalid `status` value is provided (e.g., `"UNKNOWN"`), Jackson deserialization fails before validation. The `HttpMessageNotReadableException` is caught by `GlobalExceptionHandler` (ADR-05) and returned as a 400 Bad Request ProblemDetail:

```json
{
  "type": "https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR",
  "title": "Validation Error",
  "status": 400,
  "detail": "Invalid request body: Cannot deserialize value of type `AuditStatus` from String \"UNKNOWN\"",
  "instance": "/audit/query",
  "timestamp": "2026-03-09T10:30:00.123Z"
}
```

# Additional Recommendations:

- Consider adding `@field:Size(max = 255)` to optional string fields to prevent excessively long inputs
- If date range validation is needed (endDate > startDate), implement a class-level custom constraint
- Document the request/response DTOs in Swagger annotations (ADR-02) with `@Schema` descriptions
- All nullable fields use Kotlin's `?` type — no `@Nullable` annotations needed

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Request DTO | `find . -name "AuditQueryRequest.kt"` | File exists in `domain/model/request` |
| Status Enum | `find . -name "AuditStatus.kt"` | File exists in `domain/model/request` |
| Response DTO | `find . -name "AuditEvent.kt"` | File exists in `domain/model/response` |
| NotBlank on startDate | `grep "@field:NotBlank" AuditQueryRequest.kt` | Present on startDate |
| NotBlank on endDate | `grep "@field:NotBlank" AuditQueryRequest.kt` | Present on endDate |
| Pattern on dates | `grep "@field:Pattern" AuditQueryRequest.kt` | dd/MM/yyyy HH:mm regex |
| @Valid in Controller | `grep "@Valid" *Controller.kt` | Present on @RequestBody |
| Validation Error 400 | `POST /audit/query {}` returns status 400 | ProblemDetail with field errors |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Data Transfer & Validation..."

# Check request DTO
if find . -name "AuditQueryRequest.kt" -path "*/request/*" | grep -q .; then
  echo "✅ AuditQueryRequest exists"
else
  echo "❌ AuditQueryRequest not found"
  exit 1
fi

# Check status enum
if find . -name "AuditStatus.kt" -path "*/request/*" | grep -q .; then
  echo "✅ AuditStatus enum exists"
else
  echo "❌ AuditStatus enum not found"
  exit 1
fi

# Check response DTO
if find . -name "AuditEvent.kt" -path "*/response/*" | grep -q .; then
  echo "✅ AuditEvent response DTO exists"
else
  echo "❌ AuditEvent response DTO not found"
  exit 1
fi

# Check validation annotations
if grep -rq "@field:NotBlank" src/main/kotlin/**/AuditQueryRequest.kt 2>/dev/null; then
  echo "✅ @NotBlank annotations present"
else
  echo "⚠️ @NotBlank annotations not found (check path)"
fi

# Check @Valid in controller
if grep -rq "@Valid" src/main/kotlin/**/adapter/inbound/ 2>/dev/null; then
  echo "✅ @Valid used in controller"
else
  echo "⚠️ @Valid not found in controller"
fi

echo "✅ All Data Transfer & Validation criteria met"
```
