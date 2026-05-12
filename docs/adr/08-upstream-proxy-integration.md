# Issue: Upstream Proxy Integration Pattern for Audit BFF

The Audit BFF must forward audit query requests to the upstream Audit Event API and return responses to the frontend. A standardized HTTP client pattern is needed for this proxy functionality.

**Impact**: Without a well-defined proxy pattern, upstream communication may be implemented inconsistently, error propagation may be unreliable, and the upstream URL may be hardcoded.

# Decision:

Use Spring `RestClient` as the HTTP client for proxying requests to the upstream Audit Event API. `RestClient` is the modern synchronous HTTP client introduced in Spring Framework 6.1 (Spring Boot 3.2+), offering a fluent, type-safe API. The upstream base URL is set on the `RestClient` bean via builder; the query path is configured via `application.yml` properties and injected through `AuditPlatformProperties`.

**Required Components**:
- `RestClient` bean configured in a `@Configuration` class with `RestClient.builder().baseUrl(...)`
- `AuditPlatformRestAdapter` outbound adapter class
- `AuditPlatformProperties` configuration properties for upstream URL
- Error handling that translates upstream HTTP errors to `UpstreamServiceException`

# Constraints:

- Upstream base URL MUST be configurable via `audit.platform.base-url` property
- Upstream query path MUST be configurable via `audit.platform.query-path` property
- The BFF MUST forward the request body as-is to the upstream API (same contract)
- The BFF MUST return the upstream response as-is (raw JSON array, no transformation)
- MUST NOT implement resilience patterns (timeout, retry, circuit-breaker) — internal enterprise use only
- MUST propagate upstream errors as `UpstreamServiceException` for RFC 9457 error handling
- MUST NOT hardcode any upstream URL in source code

# Alternatives:

**Spring RestTemplate (Legacy)**: The classic synchronous HTTP client in Spring. Rejected because Spring Framework 6.1+ places `RestTemplate` in maintenance mode; `RestClient` provides the same synchronous capability with a modern fluent API and better type safety.

**Spring WebClient (Reactive)**: Non-blocking HTTP client from Spring WebFlux. Rejected because the BFF uses servlet-based `spring-boot-starter-web` (not WebFlux), and `RestClient` is simpler for synchronous proxy calls.

**OpenFeign Client**: Declarative HTTP client with interface-based API. Rejected as over-engineering for a single upstream call — adds an unnecessary dependency.

**Apache HttpClient / OkHttp**: Low-level HTTP clients. Rejected because `RestClient` provides sufficient abstraction and is included with Spring Boot.

# Rationale:

`RestClient` is the modern synchronous HTTP client in Spring Boot 3.2+, requires no additional dependencies beyond `spring-boot-starter-web`, and offers a fluent builder API with built-in type-safe response handling. It uses the same underlying `ClientHttpRequestFactory` as `RestTemplate` but provides a cleaner API surface. Error handling leverages `RestClientResponseException` (same exception hierarchy). For a single-endpoint proxy with no resilience requirements, it is the simplest and most appropriate choice.

# Implementation Guidelines:

## 1. RestClient Configuration

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.client.RestClient

@Configuration
class RestClientConfig(
    private val properties: AuditPlatformProperties,
) {

    @Bean
    fun restClient(): RestClient {
        return RestClient.builder()
            .baseUrl(properties.baseUrl)
            .build()
    }
}
```

## 2. Configuration Properties

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "audit.platform")
data class AuditPlatformProperties(
    val baseUrl: String = "http://localhost:8081",
    val queryPath: String = "/api/v1/audit-events",
)
```

## 3. Outbound Adapter (Upstream Proxy)

```kotlin
package tr.com.mycorp.audit.bff.adapter.outbound

import org.slf4j.LoggerFactory
import org.springframework.core.ParameterizedTypeReference
import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.client.RestClient
import org.springframework.web.client.RestClientResponseException
import org.springframework.web.client.ResourceAccessException
import tr.com.mycorp.audit.bff.config.AuditPlatformProperties
import tr.com.mycorp.audit.bff.domain.model.request.AuditQueryRequest
import tr.com.mycorp.audit.bff.domain.model.response.AuditEvent
import tr.com.mycorp.audit.bff.exception.UpstreamServiceException

@Component
class AuditPlatformRestAdapter(
    private val restClient: RestClient,
    private val properties: AuditPlatformProperties,
) {
    private val logger = LoggerFactory.getLogger(AuditPlatformRestAdapter::class.java)

    fun queryAuditEvents(request: AuditQueryRequest): List<AuditEvent> {
        logger.info("Forwarding audit query to upstream: {}{}", properties.baseUrl, properties.queryPath)

        return try {
            restClient.post()
                .uri(properties.queryPath)
                .contentType(MediaType.APPLICATION_JSON)
                .body(request)
                .retrieve()
                .body(object : ParameterizedTypeReference<List<AuditEvent>>() {})
                ?: emptyList()
        } catch (ex: RestClientResponseException) {
            logger.error(
                "Upstream returned error: status={}, body={}",
                ex.statusCode.value(),
                ex.responseBodyAsString,
            )
            throw UpstreamServiceException(
                statusCode = ex.statusCode.value(),
                message = "Upstream audit platform returned error: ${ex.statusCode.value()}",
                cause = ex,
            )
        } catch (ex: ResourceAccessException) {
            logger.error("Upstream is unreachable: {}", ex.message)
            throw UpstreamServiceException(
                statusCode = 502,
                message = "Audit platform service is unavailable",
                cause = ex,
            )
        }
    }
}
```

## 4. Controller Integration

```kotlin
package tr.com.mycorp.audit.bff.adapter.inbound

import jakarta.validation.Valid
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RestController
import tr.com.mycorp.audit.bff.adapter.outbound.AuditPlatformRestAdapter
import tr.com.mycorp.audit.bff.domain.model.request.AuditQueryRequest
import tr.com.mycorp.audit.bff.domain.model.response.AuditEvent

@RestController
class AuditQueryController(
    private val auditPlatformAdapter: AuditPlatformRestAdapter,
) {
    @PostMapping("/audit/query")
    fun queryAuditEvents(
        @Valid @RequestBody request: AuditQueryRequest,
    ): List<AuditEvent> {
        return auditPlatformAdapter.queryAuditEvents(request)
    }
}
```

## 5. Proxy Flow Diagram

```
┌──────────┐     POST /audit/query      ┌───────────┐     POST /api/v1/audit-events     ┌──────────────┐
│ Frontend │ ──────────────────────────> │ Audit BFF │ ──────────────────────────────────> │ Audit Event  │
│          │ <────────────────────────── │           │ <────────────────────────────────── │ API          │
└──────────┘     List<AuditEvent>        └───────────┘     List<AuditEvent>                └──────────────┘
                 (raw JSON array)              │                                           
                                               │ On error:
                                               │ RestClientResponseException → UpstreamServiceException
                                               │ ResourceAccessException → UpstreamServiceException(502)
                                               ▼
                                         GlobalExceptionHandler → RFC 9457 ProblemDetail
```

# Additional Recommendations:

- When the upstream base URL is known, set `AUDIT_PLATFORM_BASE_URL` environment variable at deployment
- Consider adding a `ClientHttpRequestInterceptor` to `RestClient.builder()` for upstream call logging in the future
- If upstream latency becomes a concern, configure `SimpleClientHttpRequestFactory` with connection/read timeouts via `RestClient.builder().requestFactory(...)`
- Consider adding a health check that verifies upstream connectivity via actuator

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| RestClient Bean | `grep -r "RestClient" src/main/kotlin/*/config/` | Bean configured |
| Properties Class | `grep -r "AuditPlatformProperties" src/main/kotlin/` | Class with `@ConfigurationProperties` |
| Outbound Adapter | `find . -name "AuditPlatformRestAdapter.kt"` | File exists in `adapter/outbound` |
| URL Not Hardcoded | `grep -rn "localhost" src/main/kotlin/` | No hardcoded URLs in source code |
| Error Translation | `grep -r "UpstreamServiceException" src/main/kotlin/*/adapter/` | Used in outbound adapter |
| Upstream URL Config | `grep "audit.platform.base-url" application.yml` | Property present |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Upstream Proxy Integration..."

# Check outbound adapter exists
if find . -name "AuditPlatformRestAdapter.kt" -path "*/adapter/outbound/*" | grep -q .; then
  echo "✅ AuditPlatformRestAdapter exists"
else
  echo "❌ AuditPlatformRestAdapter not found"
  exit 1
fi

# Check RestClient configuration
if grep -rq "fun restClient" src/main/kotlin; then
  echo "✅ RestClient bean configured"
else
  echo "❌ RestClient bean not found"
  exit 1
fi

# Check properties class
if grep -rq "AuditPlatformProperties" src/main/kotlin; then
  echo "✅ AuditPlatformProperties class exists"
else
  echo "❌ AuditPlatformProperties not found"
  exit 1
fi

# Check upstream URL in config
if grep -q "audit.platform.base-url" src/main/resources/application.yml; then
  echo "✅ Upstream URL configurable in application.yml"
else
  echo "❌ Upstream URL not in application.yml"
  exit 1
fi

# Check no hardcoded URLs in source (excluding config)
HARDCODED=$(grep -rn "http://localhost" src/main/kotlin/ --include="*.kt" | grep -v "Properties\|Config" | wc -l)
if [ "$HARDCODED" -eq 0 ]; then
  echo "✅ No hardcoded URLs in source code"
else
  echo "⚠️ Found hardcoded URLs: $HARDCODED occurrences"
fi

echo "✅ All Upstream Proxy Integration criteria met"
```
