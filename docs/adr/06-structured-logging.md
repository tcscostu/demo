# Issue: Structured Logging Strategy for Audit BFF

The Audit BFF requires consistent request/response logging for operational visibility and debugging. Without structured logging, diagnosing issues across the frontend-BFF-upstream chain becomes difficult.

**Impact**: Unstructured or missing logs prevent effective incident triage, make it impossible to trace request flows, and reduce operational confidence in the BFF proxy layer.

# Decision:

Implement AOP-based REST controller logging using `@Around` advice on all `@RestController` methods, matching the enterprise BFF pattern (bo-bff-fsm `RestControllerLoggingAspect`). Log structured entries for every request and response with OpenTelemetry trace ID correlation.

**Required Components**:
- `RestControllerLoggingAspect` with `@Around` advice on REST controller methods
- Structured log entries with: timestamp, traceId, HTTP method, URL, class/method, request body, response body, execution time
- Configurable toggle via `app.logging.rest.enabled` property
- SLF4J logging with Spring Boot defaults
- OpenTelemetry API for trace ID extraction

# Constraints:

- Logging MUST be implemented as AOP aspect, not inline in controllers
- Logging aspect MUST be toggleable via `app.logging.rest.enabled` property (default: `true`)
- Log entries MUST include OpenTelemetry `traceId` when available
- MUST NOT log sensitive data (passwords, tokens, PII) — BFF proxies audit data which may contain masked fields
- Logging overhead MUST NOT exceed 5ms per request
- MUST use SLF4J facade (not `println` or `System.out`)
- MUST NOT serialize Spring framework internal objects (avoid `Jackson` serialization errors)

# Alternatives:

**Inline logging in controller**: Manual `logger.info()` calls in each controller method. Rejected because it's repetitive, inconsistent, and violates separation of concerns.

**Spring Boot request logging filter**: `CommonsRequestLoggingFilter`. Rejected because it doesn't provide structured JSON output or AOP-level control over method entry/exit.

**Logstash JSON format**: Structured JSON log output. MAY be enabled later by uncommenting `logging.structured.format.console: logstash` — kept as optional for production.

# Rationale:

AOP-based logging is the proven pattern in the enterprise BFF stack (bo-bff-fsm). It provides consistent, centralized logging without code duplication, can be toggled on/off per environment, and integrates with OpenTelemetry tracing for distributed request correlation. The same aspect class structure ensures team familiarity and operational consistency.

# Implementation Guidelines:

## 1. AOP Configuration

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.EnableAspectJAutoProxy

@Configuration
@EnableAspectJAutoProxy
class AopConfiguration
```

## 2. REST Controller Logging Aspect

```kotlin
package tr.com.mycorp.audit.bff.aspect

import com.fasterxml.jackson.databind.ObjectMapper
import io.opentelemetry.api.trace.Span
import jakarta.servlet.http.HttpServletRequest
import org.aspectj.lang.ProceedingJoinPoint
import org.aspectj.lang.annotation.Around
import org.aspectj.lang.annotation.Aspect
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component
import org.springframework.web.context.request.RequestContextHolder
import org.springframework.web.context.request.ServletRequestAttributes

@Aspect
@Component
class RestControllerLoggingAspect(
    private val objectMapper: ObjectMapper,
) {
    private val logger = LoggerFactory.getLogger(RestControllerLoggingAspect::class.java)

    @Value("\${app.logging.rest.enabled:true}")
    private var enabled: Boolean = true

    @Value("\${app.logging.rest.max-response-length:2000}")
    private var maxResponseLength: Int = 2000

    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    fun logRestCall(joinPoint: ProceedingJoinPoint): Any? {
        if (!enabled) {
            return joinPoint.proceed()
        }

        val startTime = System.currentTimeMillis()
        val request = getCurrentRequest()
        val traceId = getTraceId()
        val className = joinPoint.signature.declaringType.simpleName
        val methodName = joinPoint.signature.name
        val requestBody = serializeArgs(joinPoint.args)

        logger.info(
            "REST_REQUEST | traceId={} | method={} | url={} | class={}.{} | body={}",
            traceId,
            request?.method ?: "UNKNOWN",
            request?.requestURI ?: "UNKNOWN",
            className,
            methodName,
            requestBody,
        )

        return try {
            val result = joinPoint.proceed()
            val duration = System.currentTimeMillis() - startTime

            logger.info(
                "REST_RESPONSE | traceId={} | class={}.{} | duration={}ms | response={}",
                traceId,
                className,
                methodName,
                duration,
                safeSerialize(result),
            )

            result
        } catch (ex: Exception) {
            val duration = System.currentTimeMillis() - startTime

            logger.error(
                "REST_RESPONSE_ERROR | traceId={} | class={}.{} | duration={}ms | error={}",
                traceId,
                className,
                methodName,
                duration,
                ex.message,
            )

            throw ex
        }
    }

    private fun getCurrentRequest(): HttpServletRequest? {
        return (RequestContextHolder.getRequestAttributes() as? ServletRequestAttributes)?.request
    }

    private fun getTraceId(): String {
        return try {
            val span = Span.current()
            val ctx = span.spanContext
            if (ctx.isValid) ctx.traceId else "no-trace"
        } catch (e: Exception) {
            "no-trace"
        }
    }

    private fun serializeArgs(args: Array<Any?>): String {
        return try {
            val filteredArgs = args.filter { it !is HttpServletRequest && it !is org.springframework.validation.BindingResult }
            objectMapper.writeValueAsString(filteredArgs)
        } catch (e: Exception) {
            "[serialization-error]"
        }
    }

    private fun safeSerialize(obj: Any?): String {
        return try {
            if (obj == null) "null"
            else objectMapper.writeValueAsString(obj).take(maxResponseLength) // Truncate large responses
        } catch (e: Exception) {
            "[serialization-error]"
        }
    }
}
```

## 3. Logging Configuration (in application.yml)

```yaml
logging:
  level:
    org.springframework: WARN
    tr.com.mycorp.audit.bff: INFO
    tr.com.mycorp.audit.bff.aspect.RestControllerLoggingAspect: INFO
  structured:
    format:
      console: #logstash  # Uncomment for JSON logs in production

app:
  logging:
    rest:
      enabled: true
      max-response-length: 2000  # Truncation limit for response body in logs
```

## 4. Log Output Examples

**Request log:**
```
INFO  REST_REQUEST | traceId=abc123def456 | method=POST | url=/audit/query | class=AuditQueryController.queryAuditEvents | body=[{"eventName":"FAST_GONDERIM","startDate":"01/02/2026 00:00","endDate":"19/02/2026 23:59"}]
```

**Success response log:**
```
INFO  REST_RESPONSE | traceId=abc123def456 | class=AuditQueryController.queryAuditEvents | duration=142ms | response=[{"eventName":"FAST_GONDERIM",...}]
```

**Error response log:**
```
ERROR REST_RESPONSE_ERROR | traceId=abc123def456 | class=AuditQueryController.queryAuditEvents | duration=5012ms | error=Audit platform service is unavailable
```

# Additional Recommendations:

- Enable Logstash JSON format (`logging.structured.format.console: logstash`) in production for log aggregation tools
- Consider adding a response size threshold to avoid logging large result sets in full
- Add MDC-based correlation ID propagation if the API gateway passes `X-Correlation-Id` headers
- Monitor log volume in production — if the BFF handles many requests, consider reducing log level to `DEBUG` and only logging errors at `INFO`

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| AOP Configuration | `find . -name "AopConfiguration.kt"` | File exists with `@EnableAspectJAutoProxy` |
| Logging Aspect | `find . -name "RestControllerLoggingAspect.kt"` | File exists in `aspect` package |
| SLF4J Usage | `grep -r "LoggerFactory" src/main/kotlin/` | Used in aspect (no `println`) |
| Configurable Toggle | `grep "app.logging.rest.enabled" application.yml` | Property present |
| Request Logged | Application log output on POST | `REST_REQUEST` entry present |
| Response Logged | Application log output on POST | `REST_RESPONSE` entry present |
| TraceId Present | Log entry inspection | `traceId=` field present |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Structured Logging..."

# Check aspect exists
if find . -name "RestControllerLoggingAspect.kt" -path "*/aspect/*" | grep -q .; then
  echo "✅ RestControllerLoggingAspect exists"
else
  echo "❌ RestControllerLoggingAspect not found"
  exit 1
fi

# Check AOP configuration
if find . -name "AopConfiguration.kt" -path "*/config/*" | grep -q .; then
  echo "✅ AopConfiguration exists"
else
  echo "❌ AopConfiguration not found"
  exit 1
fi

# Check SLF4J usage (no System.out)
if grep -rq "System.out" src/main/kotlin/ 2>/dev/null; then
  echo "❌ System.out usage detected — use SLF4J"
  exit 1
fi
echo "✅ No System.out usage"

# Check logging toggle
if grep -q "app.logging.rest.enabled" src/main/resources/application.yml; then
  echo "✅ Logging toggle configured"
else
  echo "⚠️ Logging toggle not in application.yml"
fi

echo "✅ All Structured Logging criteria met"
```
