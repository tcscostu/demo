# Issue: Application Configuration Management for Audit BFF

The Audit BFF requires centralized configuration management with environment-specific profiles and externalized upstream service URL. Without standardized configuration, deployment becomes error-prone and environment-specific adjustments require code changes.

**Impact**: Hardcoded configuration values or inconsistent configuration patterns lead to deployment failures, environment-specific bugs, and inability to configure upstream services at deployment time.

# Decision:

Implement `application.yml` as the centralized configuration file with environment variable substitution for deployment-time values. The upstream audit platform URL MUST be configurable via application properties.

**Required Configuration Areas**:
- Server port and context path
- Upstream audit platform base URL (configurable, not hardcoded)
- SpringDoc/Swagger UI settings
- Actuator endpoint exposure
- Logging levels
- CORS allowed origins
- AOP logging toggle

# Constraints:

- MUST use `application.yml` format (consistent with bo-bff-fsm)
- MUST NOT include any database configuration (no datasource, JPA, or Hibernate)
- MUST NOT include any security/JWT configuration (authentication handled by API gateway)
- Upstream base URL MUST be configurable via environment variable with a placeholder default
- CORS origins MUST be configurable for different environments
- Sensitive values MUST use environment variable substitution (`${VAR:default}`)

# Alternatives:

**application.properties**: Flat key-value format. Rejected because `application.yml` is used in bo-bff-fsm and provides better readability for nested configuration.

**Spring Cloud Config Server**: External configuration server. Rejected as over-engineering for a simple internal BFF — environment variables are sufficient.

**Multiple profile-specific files**: Separate `application-dev.yml`, `application-prod.yml`. MAY be added later if needed; single file with profile sections is sufficient for now.

# Rationale:

A single `application.yml` with environment variable substitution provides a clean, readable configuration that can be customized per environment via standard deployment mechanisms (Kubernetes ConfigMaps, Docker environment variables, shell exports). This matches the bo-bff-fsm pattern while removing all database and security configuration that is out of scope.

# Implementation Guidelines:

## Complete application.yml

```yaml
server:
  port: ${SERVER_PORT:8080}
  shutdown: graceful
  servlet:
    context-path: /

spring:
  application:
    name: bff-audit
  output:
    ansi:
      enabled: never

# ─── Upstream Audit Platform Configuration ───
audit:
  platform:
    base-url: ${AUDIT_PLATFORM_BASE_URL:http://localhost:8081}
    query-path: ${AUDIT_PLATFORM_QUERY_PATH:/api/v1/audit-events}

# ─── Actuator ───
management:
  endpoints:
    web:
      exposure:
        include: "*"
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true

# ─── Logging ───
logging:
  level:
    org.springframework: WARN
    tr.com.mycorp.audit.bff: INFO
    tr.com.mycorp.audit.bff.aspect.RestControllerLoggingAspect: INFO
  structured:
    format:
      console: #logstash

# ─── SpringDoc / Swagger UI ───
springdoc:
  api-docs:
    enabled: true
    path: /v3/api-docs
  swagger-ui:
    enabled: true
    path: /
    operations-sorter: alpha
    tags-sorter: alpha
    try-it-out-enabled: true
    filter: true
    display-request-duration: true
  show-actuator: true

# ─── AOP Logging ───
app:
  logging:
    rest:
      enabled: true
      max-response-length: 2000  # Truncation limit for response body in logs

# ─── CORS ───
tr:
  com:
    mycorp:
      audit:
        bff:
          cors:
            allowed-origins:
              - ${CORS_ORIGIN_1:http://localhost:3000}
              - ${CORS_ORIGIN_2:http://localhost:3017}
```

## Configuration Properties Binding (Kotlin)

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "audit.platform")
data class AuditPlatformProperties(
    val baseUrl: String = "http://localhost:8081",
    val queryPath: String = "/api/v1/audit-events",
)
```

## CORS Configuration (Kotlin)

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "tr.com.mycorp.audit.bff.cors")
data class CorsProperties(
    val allowedOrigins: List<String> = emptyList(),
)
```

```kotlin
package tr.com.mycorp.audit.bff.config

import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.CorsRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class WebConfig(
    private val corsProperties: CorsProperties,
) : WebMvcConfigurer {

    override fun addCorsMappings(registry: CorsRegistry) {
        registry.addMapping("/**")
            .allowedOrigins(*corsProperties.allowedOrigins.toTypedArray())
            .allowedMethods("POST", "GET", "OPTIONS")
            .allowedHeaders("Content-Type", "Accept")
    }
}
```

Enable configuration properties scanning in the application:

```kotlin
@SpringBootApplication
@ConfigurationPropertiesScan
class BffAuditApplication
```

## Environment Variables Reference

| Variable | Purpose | Default | Required in Prod |
|----------|---------|---------|------------------|
| `SERVER_PORT` | HTTP server port | `8080` | No |
| `AUDIT_PLATFORM_BASE_URL` | Upstream audit API base URL | `http://localhost:8081` | **Yes** |
| `AUDIT_PLATFORM_QUERY_PATH` | Upstream query endpoint path | `/api/v1/audit-events` | No |
| `CORS_ORIGIN_1` | First allowed CORS origin | `http://localhost:3000` | Yes |
| `CORS_ORIGIN_2` | Second allowed CORS origin | `http://localhost:3017` | No |

# Additional Recommendations:

- When the actual upstream audit platform URL is known, update the `AUDIT_PLATFORM_BASE_URL` environment variable in the deployment configuration
- Consider adding profile-specific configuration (`spring.profiles.active`) if dev/staging/prod environments diverge significantly
- Add `spring.jackson.serialization.write-dates-as-timestamps=false` if date serialization issues arise

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| application.yml Exists | `test -f src/main/resources/application.yml` | File exists |
| Upstream URL Configurable | `grep "audit.platform.base-url" application.yml` | Property present with env var |
| No Database Config | `grep -c "datasource\|jpa\|hibernate" application.yml` | `0` |
| No Security Config | `grep -c "jwt\|security.users" application.yml` | `0` |
| Swagger UI Config | `grep "swagger-ui" application.yml` | Configuration present |
| Actuator Config | `grep "actuator" application.yml` | Configuration present |
| Properties Class | Kotlin class with `@ConfigurationProperties` | Compiles and binds |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Application Configuration..."

CONFIG="src/main/resources/application.yml"

if [ ! -f "$CONFIG" ]; then
  echo "❌ application.yml not found"
  exit 1
fi
echo "✅ application.yml exists"

# Check upstream URL configuration
if grep -q "AUDIT_PLATFORM_BASE_URL" "$CONFIG"; then
  echo "✅ Upstream base URL is configurable via environment variable"
else
  echo "❌ Upstream base URL not configurable"
  exit 1
fi

# Verify no database configuration
if grep -qE "datasource|jpa|hibernate" "$CONFIG"; then
  echo "❌ Database configuration found — must be removed"
  exit 1
fi
echo "✅ No database configuration"

# Verify no security configuration
if grep -qE "jwt|security.users" "$CONFIG"; then
  echo "❌ Security configuration found — must be removed"
  exit 1
fi
echo "✅ No security configuration"

echo "✅ All Application Configuration criteria met"
```
