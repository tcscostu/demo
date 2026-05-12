# Issue: REST API Error Handling for Audit BFF

The Audit BFF requires standardized error responses for validation failures, upstream service errors, and unexpected exceptions. Without consistent error formatting, frontend developers cannot reliably parse and display error messages.

**Impact**: Inconsistent error responses cause frontend parsing failures, unclear error messages for end users, and difficulty diagnosing issues across the BFF-to-upstream chain.

# Decision:

Implement RFC 9457 (Problem Details for HTTP APIs) compliant error handling using Spring Boot's built-in `ProblemDetail` class and `@RestControllerAdvice` global exception handlers. This aligns with the platform ADR-16 error handling strategy while using Spring Boot's native implementation.

**Required Components**:
- `@RestControllerAdvice` global exception handler returning `ProblemDetail`
- Content-Type `application/problem+json` for all error responses
- Custom exception classes: `UpstreamServiceException`, `ValidationException`
- HTTP status code mapping per error category
- Timestamp and instance URI in extension fields

# Constraints:

- All error responses MUST use RFC 9457 Problem Details format
- Content-Type MUST be `application/problem+json` for error responses
- Problem Details fields MUST NOT contain sensitive information (stack traces, internal IPs)
- Error responses MUST include `type`, `title`, `status`, `detail`, `instance`, and `timestamp`
- HTTP status codes MUST follow RFC 7231 semantic meanings
- Error handling MUST NOT introduce blocking operations or significant latency (< 2ms overhead)

# Alternatives:

**Custom error JSON format**: Ad-hoc `{"error": "...", "code": ...}` structure. Rejected because it diverges from the platform RFC 9457 standard (ADR-16) and reduces interoperability.

**Spring Boot default error format**: `DefaultErrorAttributes` with `/error` endpoint. Rejected because it doesn't produce RFC 9457 compliant output and exposes internal details.

**Reactive exception mapper (Quarkus-style)**: Not applicable — the BFF uses Spring Boot servlet stack, not Quarkus/Mutiny.

# Rationale:

Spring Boot 3.x natively supports RFC 9457 via the `ProblemDetail` class (from Spring Framework 6+), making implementation straightforward without additional libraries. Using `@RestControllerAdvice` matches the bo-bff-fsm pattern (`GlobalExceptionHandlerConfig`). The standard format ensures frontend developers can write a single error parsing logic for all error types.

# Implementation Guidelines:

## 1. Custom Exception Classes

```kotlin
package tr.com.mycorp.audit.bff.exception

/**
 * Thrown when the upstream audit platform returns an error or is unreachable.
 */
class UpstreamServiceException(
    val statusCode: Int,
    override val message: String,
    cause: Throwable? = null,
) : RuntimeException(message, cause)

/**
 * Thrown when request validation fails beyond Jakarta Bean Validation.
 */
class ValidationException(
    override val message: String,
    val fieldErrors: Map<String, String> = emptyMap(),
) : RuntimeException(message)
```

## 2. Global Exception Handler

```kotlin
package tr.com.mycorp.audit.bff.exception

import jakarta.servlet.http.HttpServletRequest
import org.slf4j.LoggerFactory
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.converter.HttpMessageNotReadableException
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.net.URI
import java.time.Instant

@RestControllerAdvice
class GlobalExceptionHandler {

    private val logger = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)

    /**
     * Handles Jakarta Bean Validation errors (e.g., @NotBlank, @Pattern).
     */
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(
        ex: MethodArgumentNotValidException,
        request: HttpServletRequest,
    ): ProblemDetail {
        val fieldErrors = ex.bindingResult.fieldErrors.joinToString("; ") {
            "${it.field}: ${it.defaultMessage}"
        }
        logger.warn("Validation error: {}", fieldErrors)

        return ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            fieldErrors
        ).apply {
            type = URI.create("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR")
            title = "Validation Error"
            instance = URI.create(request.requestURI)
            setProperty("timestamp", Instant.now().toString())
        }
    }

    /**
     * Handles Jackson deserialization errors (e.g., invalid enum value, malformed JSON).
     */
    @ExceptionHandler(HttpMessageNotReadableException::class)
    fun handleMessageNotReadable(
        ex: HttpMessageNotReadableException,
        request: HttpServletRequest,
    ): ProblemDetail {
        logger.warn("Request parsing error: {}", ex.message)

        return ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "Invalid request body: ${ex.mostSpecificCause.message}"
        ).apply {
            type = URI.create("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR")
            title = "Validation Error"
            instance = URI.create(request.requestURI)
            setProperty("timestamp", Instant.now().toString())
        }
    }

    /**
     * Handles custom validation exceptions.
     */
    @ExceptionHandler(ValidationException::class)
    fun handleCustomValidation(
        ex: ValidationException,
        request: HttpServletRequest,
    ): ProblemDetail {
        logger.warn("Custom validation error: {}", ex.message)

        return ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            ex.message
        ).apply {
            type = URI.create("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR")
            title = "Validation Error"
            instance = URI.create(request.requestURI)
            setProperty("timestamp", Instant.now().toString())
            if (ex.fieldErrors.isNotEmpty()) {
                setProperty("fieldErrors", ex.fieldErrors)
            }
        }
    }

    /**
     * Handles upstream audit platform errors.
     */
    @ExceptionHandler(UpstreamServiceException::class)
    fun handleUpstreamError(
        ex: UpstreamServiceException,
        request: HttpServletRequest,
    ): ProblemDetail {
        logger.error("Upstream service error: {} - {}", ex.statusCode, ex.message)

        val status = when {
            ex.statusCode in 400..499 -> HttpStatus.valueOf(ex.statusCode)
            else -> HttpStatus.BAD_GATEWAY
        }

        return ProblemDetail.forStatusAndDetail(
            status,
            ex.message
        ).apply {
            type = URI.create("https://api.audit-bff.mycorp.com.tr/problems/UPSTREAM_ERROR")
            title = "Upstream Service Error"
            instance = URI.create(request.requestURI)
            setProperty("timestamp", Instant.now().toString())
        }
    }

    /**
     * Catches all unexpected exceptions.
     */
    @ExceptionHandler(Exception::class)
    fun handleGenericException(
        ex: Exception,
        request: HttpServletRequest,
    ): ProblemDetail {
        logger.error("Unexpected error", ex)

        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred. Please try again later."
        ).apply {
            type = URI.create("https://api.audit-bff.mycorp.com.tr/problems/INTERNAL_ERROR")
            title = "Internal Server Error"
            instance = URI.create(request.requestURI)
            setProperty("timestamp", Instant.now().toString())
        }
    }
}
```

## 3. HTTP Status Code Mapping

| Error Category | HTTP Status | Problem Type Suffix |
|----------------|-------------|---------------------|
| Missing/invalid request fields | 400 Bad Request | `VALIDATION_ERROR` |
| Invalid date format | 400 Bad Request | `VALIDATION_ERROR` |
| Invalid enum value (status) | 400 Bad Request | `VALIDATION_ERROR` |
| Malformed JSON body | 400 Bad Request | `VALIDATION_ERROR` |
| Upstream returns 4xx | Propagated status | `UPSTREAM_ERROR` |
| Upstream returns 5xx | 502 Bad Gateway | `UPSTREAM_ERROR` |
| Upstream unreachable | 502 Bad Gateway | `UPSTREAM_ERROR` |
| Unexpected BFF error | 500 Internal Server Error | `INTERNAL_ERROR` |

## 4. Example Error Responses

**Validation Error (400):**
```json
{
  "type": "https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR",
  "title": "Validation Error",
  "status": 400,
  "detail": "startDate: must not be blank; endDate: must not be blank",
  "instance": "/audit/query",
  "timestamp": "2026-03-09T10:30:00.123Z"
}
```

**Upstream Error (502):**
```json
{
  "type": "https://api.audit-bff.mycorp.com.tr/problems/UPSTREAM_ERROR",
  "title": "Upstream Service Error",
  "status": 502,
  "detail": "Audit platform service is unavailable",
  "instance": "/audit/query",
  "timestamp": "2026-03-09T10:30:00.123Z"
}
```

# Additional Recommendations:

- Consider adding a `correlationId` extension field from the request header for end-to-end tracing
- Log all errors with sufficient context for debugging but never log sensitive data
- In production, monitor error rates via actuator metrics (`http.server.requests` with `status` tag)
- Consider adding `retryable` boolean field to help frontend decide whether to auto-retry

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Global Exception Handler | `find . -name "GlobalExceptionHandler.kt"` | File exists |
| Custom Exceptions | `find . -name "*Exception.kt" -path "*exception*"` | At least 2 files |
| ProblemDetail Used | `grep -r "ProblemDetail" src/` | Used in exception handler |
| Validation Error Returns 400 | Send invalid request | RFC 9457 with status 400 |
| Upstream Error Returns 502 | Mock upstream failure | RFC 9457 with status 502 |
| No Stack Traces in Response | Inspect error response body | No stack trace or internal paths |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating REST API Error Handling..."

# Check exception handler exists
if find . -name "GlobalExceptionHandler.kt" -path "*/exception/*" | grep -q .; then
  echo "✅ GlobalExceptionHandler exists"
else
  echo "❌ GlobalExceptionHandler not found"
  exit 1
fi

# Check ProblemDetail usage
if grep -rq "ProblemDetail" src/main/kotlin; then
  echo "✅ ProblemDetail class used"
else
  echo "❌ ProblemDetail not used"
  exit 1
fi

# Check custom exceptions
EXCEPTION_COUNT=$(find . -name "*Exception.kt" -path "*/exception/*" | wc -l)
if [ "$EXCEPTION_COUNT" -ge 2 ]; then
  echo "✅ Custom exception classes found: $EXCEPTION_COUNT"
else
  echo "❌ Insufficient custom exception classes: $EXCEPTION_COUNT"
  exit 1
fi

# Runtime validation (requires running application)
RESPONSE=$(curl -s -w "\n%{http_code}" -X POST http://localhost:8080/audit/query \
  -H "Content-Type: application/json" -d '{}' 2>/dev/null)
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -n -1)

if [ "$HTTP_CODE" = "400" ] && echo "$BODY" | grep -q "VALIDATION_ERROR"; then
  echo "✅ Validation error returns 400 with Problem Details"
fi

echo "✅ All REST API Error Handling criteria met"
```
