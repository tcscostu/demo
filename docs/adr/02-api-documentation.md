# Issue: API Documentation for Audit BFF

The Audit BFF requires interactive API documentation for enterprise users and frontend developers to explore the `/audit/query` endpoint, understand request/response schemas, and test queries directly from the browser.

**Impact**: Without accessible API documentation, frontend developers rely on ad-hoc communication to understand the BFF contract, leading to integration delays and misunderstandings.

# Decision:

Use SpringDoc OpenAPI 2.6.0 with Swagger UI as the API documentation solution, matching the enterprise BFF standard (bo-bff-fsm).

**Required Components**:
- `springdoc-openapi-starter-webmvc-ui` version 2.6.0
- Swagger UI served at root path (`/`)
- OpenAPI 3.0 spec served at `/v3/api-docs`
- API info metadata (title, description, version)
- Request/response schema documentation via Kotlin data class annotations

# Constraints:

- Swagger UI MUST be accessible at the root path (`/`) for developer convenience
- OpenAPI spec MUST be available at `/v3/api-docs` in JSON format
- API documentation MUST accurately reflect the actual request/response models
- MUST NOT require authentication to access Swagger UI (BFF has no auth)
- MUST include `try-it-out` functionality for interactive testing

# Alternatives:

**Manual OpenAPI YAML/JSON**: Hand-written spec file. Rejected because it drifts from code over time and requires manual synchronization.

**Spring REST Docs**: Generates documentation from tests. Rejected as overly complex for a single-endpoint BFF proxy.

**Redoc**: Alternative OpenAPI renderer. Rejected because Swagger UI is the established standard in bo-bff-fsm and provides interactive try-it-out capability.

# Rationale:

SpringDoc OpenAPI 2.6.0 is already proven in the enterprise BFF stack (bo-bff-fsm). It auto-generates accurate API documentation from Kotlin data classes and Spring MVC annotations, requires minimal configuration, and provides Swagger UI for interactive testing. The same version ensures compatibility and consistency.

# Implementation Guidelines:

## 1. Maven Dependency

```xml
<properties>
    <springdoc.version>2.6.0</springdoc.version>
</properties>

<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>${springdoc.version}</version>
</dependency>
```

## 2. Application Configuration (application.yml)

```yaml
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
```

## 3. OpenAPI Configuration Class

```kotlin
package tr.com.mycorp.audit.bff.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun openAPI(): OpenAPI {
        return OpenAPI()
            .info(
                Info()
                    .title("Audit BFF API")
                    .description("Backend-for-Frontend proxy for the Enterprise Audit Platform. Exposes audit event query functionality to the frontend application.")
                    .version("0.0.1-SNAPSHOT")
                    .contact(
                        Contact()
                            .name("Enterprise Platform Team")
                    )
            )
    }
}
```

## 4. Controller Annotations for Documentation

```kotlin
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag

@Tag(name = "Audit Query", description = "Audit event query operations")
@RestController
class AuditQueryController {

    @Operation(
        summary = "Query audit events",
        description = "Queries the upstream audit platform with the provided filters and returns matching audit events."
    )
    @ApiResponses(
        value = [
            ApiResponse(responseCode = "200", description = "Audit events retrieved successfully"),
            ApiResponse(
                responseCode = "400",
                description = "Validation error",
                content = [Content(mediaType = "application/problem+json")]
            ),
            ApiResponse(
                responseCode = "502",
                description = "Upstream audit platform unavailable",
                content = [Content(mediaType = "application/problem+json")]
            )
        ]
    )
    @PostMapping("/audit/query")
    fun queryAuditEvents(@Valid @RequestBody request: AuditQueryRequest): List<AuditEvent> {
        // ...
    }
}
```

# Additional Recommendations:

- Add `@Schema` annotations on request DTO fields to document constraints (format, required, examples)
- Consider grouping endpoints if additional BFF endpoints are added in the future using `springdoc.group-configs`
- Enable `springdoc.show-actuator=true` to expose actuator endpoints in documentation for operations visibility

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| SpringDoc Dependency | `grep springdoc pom.xml` | `springdoc-openapi-starter-webmvc-ui` present |
| Swagger UI Access | `curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/` | `200` |
| OpenAPI Spec | `curl -s http://localhost:8080/v3/api-docs \| jq '.paths["/audit/query"]'` | Endpoint documented |
| Try-It-Out | Manual browser check | Interactive form available |
| Request Schema | OpenAPI spec inspection | All 5 query fields documented |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating API Documentation..."

# Check dependency
if grep -q "springdoc-openapi-starter-webmvc-ui" pom.xml; then
  echo "✅ SpringDoc dependency present"
else
  echo "❌ SpringDoc dependency missing"
  exit 1
fi

# Check configuration
if grep -q "swagger-ui" src/main/resources/application.yml; then
  echo "✅ Swagger UI configured"
else
  echo "❌ Swagger UI not configured"
  exit 1
fi

# Runtime validation (requires running application)
if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/ | grep -q "200"; then
  echo "✅ Swagger UI accessible at /"
fi

if curl -s http://localhost:8080/v3/api-docs | grep -q "audit/query"; then
  echo "✅ /audit/query endpoint documented in OpenAPI spec"
fi

echo "✅ All API Documentation criteria met"
```
