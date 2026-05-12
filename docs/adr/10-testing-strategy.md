# Issue: Testing Strategy for Audit BFF

The Audit BFF requires a clear testing approach that covers the inbound controller, outbound adapter, exception handler, and validation logic to ensure correctness before deployment.

**Impact**: Without a defined testing strategy and examples, developers may write inconsistent tests, skip edge cases, or use inappropriate test frameworks.

# Decision:

Use JUnit 5 as the test framework with MockK for mocking Kotlin classes, and Spring Boot's `MockMvc` for controller integration tests. Tests follow a structured naming convention and cover the main interaction paths of the proxy BFF.

**Required Components**:
- `spring-boot-starter-test` (JUnit 5, MockMvc)
- `mockk-jvm` 1.13.8 for Kotlin mock support
- Unit tests for `AuditPlatformRestAdapter` (outbound adapter)
- Integration tests for `AuditQueryController` (inbound adapter)
- Tests for `GlobalExceptionHandler` error scenarios

# Constraints:

- MUST use JUnit 5 (JUnit Platform, included in `spring-boot-starter-test`)
- MUST use MockK (not Mockito) for mocking — consistent with Kotlin idioms and bo-bff-fsm
- Controller tests MUST use `@WebMvcTest` + `MockMvc` for HTTP-level testing
- Outbound adapter tests MUST use `MockRestServiceServer` bound to `RestClient.Builder` — no real HTTP calls
- Test classes MUST reside in `src/test/kotlin` mirroring the main source package structure
- Test method naming: `should <expected behavior> when <condition>` (lowercase, backtick-enclosed)

# Alternatives:

**Mockito**: Java mocking library. Rejected — MockK integrates better with Kotlin (handles `final` classes, coroutines, data classes natively).

**TestRestTemplate / WebTestClient**: Full server integration tests. Rejected for unit tests — `MockMvc` is lighter and sufficient for testing controller behavior without starting a full server. `MockRestServiceServer` is used for outbound adapter tests.

**Testcontainers**: Container-based integration tests. Not applicable — the BFF has no database or external dependencies to containerize.

# Rationale:

MockK + MockMvc is the standard Kotlin-idiomatic testing combination used in bo-bff-fsm. `@WebMvcTest` loads only the web layer, making tests fast and focused. MockK handles final Kotlin classes transparently without extra configuration.

# Implementation Guidelines:

## 1. Test Dependencies (already in pom.xml — ADR-01)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.jetbrains.kotlin</groupId>
    <artifactId>kotlin-test-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.mockk</groupId>
    <artifactId>mockk-jvm</artifactId>
    <version>1.13.8</version>
    <scope>test</scope>
</dependency>
```

## 2. Controller Test (Inbound Adapter)

```kotlin
package tr.com.mycorp.audit.bff.adapter.inbound

import com.fasterxml.jackson.databind.ObjectMapper
import com.ninjasquad.springmockk.MockkBean
import io.mockk.every
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.post
import tr.com.mycorp.audit.bff.adapter.outbound.AuditPlatformRestAdapter
import tr.com.mycorp.audit.bff.domain.model.request.AuditQueryRequest
import tr.com.mycorp.audit.bff.domain.model.response.AuditEvent
import tr.com.mycorp.audit.bff.domain.model.response.Outcome

@WebMvcTest(AuditQueryController::class)
class AuditQueryControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @MockkBean
    private lateinit var auditPlatformAdapter: AuditPlatformRestAdapter

    @Test
    fun `should return audit events when valid request is sent`() {
        val request = AuditQueryRequest(
            eventName = "FAST_GONDERIM",
            startDate = "01/02/2026 00:00",
            endDate = "19/02/2026 23:59",
        )
        val events = listOf(
            AuditEvent(
                eventName = "FAST_GONDERIM",
                outcome = Outcome(status = "SUCCESS", statusCode = "1000"),
            ),
        )
        every { auditPlatformAdapter.queryAuditEvents(any()) } returns events

        mockMvc.post("/audit/query") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isOk() }
            jsonPath("$[0].eventName") { value("FAST_GONDERIM") }
            jsonPath("$[0].outcome.status") { value("SUCCESS") }
        }
    }

    @Test
    fun `should return 400 when startDate is missing`() {
        val body = """{"endDate": "19/02/2026 23:59"}"""

        mockMvc.post("/audit/query") {
            contentType = MediaType.APPLICATION_JSON
            content = body
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.type") { value("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR") }
            jsonPath("$.title") { value("Validation Error") }
            jsonPath("$.status") { value(400) }
        }
    }

    @Test
    fun `should return 400 when date format is invalid`() {
        val body = """{"startDate": "2026-02-01", "endDate": "2026-02-19"}"""

        mockMvc.post("/audit/query") {
            contentType = MediaType.APPLICATION_JSON
            content = body
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.type") { value("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR") }
        }
    }

    @Test
    fun `should return 400 when status value is invalid`() {
        val body = """{"startDate": "01/02/2026 00:00", "endDate": "19/02/2026 23:59", "status": "UNKNOWN"}"""

        mockMvc.post("/audit/query") {
            contentType = MediaType.APPLICATION_JSON
            content = body
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.type") { value("https://api.audit-bff.mycorp.com.tr/problems/VALIDATION_ERROR") }
        }
    }

    @Test
    fun `should return empty list when upstream returns no results`() {
        val request = AuditQueryRequest(
            startDate = "01/02/2026 00:00",
            endDate = "19/02/2026 23:59",
        )
        every { auditPlatformAdapter.queryAuditEvents(any()) } returns emptyList()

        mockMvc.post("/audit/query") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isOk() }
            jsonPath("$") { isArray() }
            jsonPath("$.length()") { value(0) }
        }
    }
}
```

## 3. Outbound Adapter Test

```kotlin
package tr.com.mycorp.audit.bff.adapter.outbound

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.http.HttpMethod
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.test.web.client.MockRestServiceServer
import org.springframework.test.web.client.match.MockRestRequestMatchers.method
import org.springframework.test.web.client.match.MockRestRequestMatchers.requestTo
import org.springframework.test.web.client.response.MockRestResponseCreators.withServerError
import org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess
import org.springframework.web.client.RestClient
import tr.com.mycorp.audit.bff.config.AuditPlatformProperties
import tr.com.mycorp.audit.bff.domain.model.request.AuditQueryRequest
import tr.com.mycorp.audit.bff.exception.UpstreamServiceException

class AuditPlatformRestAdapterTest {

    private val properties = AuditPlatformProperties(
        baseUrl = "http://localhost:8081",
        queryPath = "/api/v1/audit-events",
    )

    private lateinit var mockServer: MockRestServiceServer
    private lateinit var adapter: AuditPlatformRestAdapter

    @BeforeEach
    fun setUp() {
        val restClientBuilder = RestClient.builder()
        mockServer = MockRestServiceServer.bindTo(restClientBuilder).build()
        val restClient = restClientBuilder.baseUrl(properties.baseUrl).build()
        adapter = AuditPlatformRestAdapter(restClient, properties)
    }

    @Test
    fun `should return audit events from upstream`() {
        val request = AuditQueryRequest(startDate = "01/02/2026 00:00", endDate = "19/02/2026 23:59")

        mockServer.expect(requestTo("http://localhost:8081/api/v1/audit-events"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(
                withSuccess(
                    """[{"eventName":"FAST_GONDERIM"}]""",
                    MediaType.APPLICATION_JSON,
                ),
            )

        val result = adapter.queryAuditEvents(request)

        assertEquals(1, result.size)
        assertEquals("FAST_GONDERIM", result[0].eventName)
        mockServer.verify()
    }

    @Test
    fun `should throw UpstreamServiceException when upstream returns error`() {
        val request = AuditQueryRequest(startDate = "01/02/2026 00:00", endDate = "19/02/2026 23:59")

        mockServer.expect(requestTo("http://localhost:8081/api/v1/audit-events"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(withServerError())

        val exception = assertThrows(UpstreamServiceException::class.java) {
            adapter.queryAuditEvents(request)
        }
        assertEquals(500, exception.statusCode)
        mockServer.verify()
    }

    @Test
    fun `should return empty list when upstream returns empty array`() {
        val request = AuditQueryRequest(startDate = "01/02/2026 00:00", endDate = "19/02/2026 23:59")

        mockServer.expect(requestTo("http://localhost:8081/api/v1/audit-events"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(withSuccess("[]", MediaType.APPLICATION_JSON))

        val result = adapter.queryAuditEvents(request)

        assertEquals(0, result.size)
        mockServer.verify()
    }
}
```

## 4. Test Package Structure

```
src/test/kotlin/tr/com/mycorp/audit/bff/
├── adapter/
│   ├── inbound/
│   │   └── AuditQueryControllerTest.kt
│   └── outbound/
│       └── AuditPlatformRestAdapterTest.kt
└── exception/
    └── GlobalExceptionHandlerTest.kt    # Optional: dedicated exception handler tests
```

## 5. Test Naming Convention

Use backtick-enclosed descriptive names following `should ... when ...` pattern:

```kotlin
@Test
fun `should return 400 when startDate is missing`() { ... }

@Test
fun `should throw UpstreamServiceException when upstream returns error`() { ... }
```

## 6. Additional Test Dependency

For `@WebMvcTest` with MockK, add `springmockk` to avoid Mockito auto-configuration conflicts:

```xml
<dependency>
    <groupId>com.ninja-squad</groupId>
    <artifactId>springmockk</artifactId>
    <version>4.0.2</version>
    <scope>test</scope>
</dependency>
```

> **Note**: This dependency replaces `@MockBean` (Mockito) with `@MockkBean` (MockK) in Spring Boot test slices.

# Additional Recommendations:

- Add `GlobalExceptionHandlerTest` if error response format verification becomes critical
- Consider adding a smoke test that starts the full application context (`@SpringBootTest`) to verify wiring
- Use `@Nested` inner classes to group related test scenarios (e.g., validation tests, upstream error tests)
- Run tests with JaCoCo coverage (ADR-07) and aim for > 80% on adapter and exception handler code

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Controller Test | `find . -name "AuditQueryControllerTest.kt"` | File exists in `adapter/inbound` |
| Adapter Test | `find . -name "AuditPlatformRestAdapterTest.kt"` | File exists in `adapter/outbound` |
| MockK Used | `grep -r "mockk" src/test/kotlin/` | MockK used for mocking |
| MockMvc Used | `grep -r "MockMvc" src/test/kotlin/` | MockMvc used in controller tests |
| springmockk Dependency | `grep "springmockk" pom.xml` | Dependency present |
| All Tests Pass | `./mvnw test` | Exit code 0 |
| Coverage Report | `test -f target/site/jacoco/index.html` | JaCoCo report generated |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Testing Strategy..."

# Check controller test
if find . -name "AuditQueryControllerTest.kt" -path "*/adapter/inbound/*" | grep -q .; then
  echo "✅ AuditQueryControllerTest exists"
else
  echo "❌ AuditQueryControllerTest not found"
  exit 1
fi

# Check adapter test
if find . -name "AuditPlatformRestAdapterTest.kt" -path "*/adapter/outbound/*" | grep -q .; then
  echo "✅ AuditPlatformRestAdapterTest exists"
else
  echo "❌ AuditPlatformRestAdapterTest not found"
  exit 1
fi

# Check MockK usage
if grep -rq "mockk" src/test/kotlin 2>/dev/null; then
  echo "✅ MockK used in tests"
else
  echo "⚠️ MockK not found in tests"
fi

# Run tests
./mvnw test -q 2>/dev/null && echo "✅ All tests pass" || echo "⚠️ Test failures"

echo "✅ All Testing Strategy criteria met"
```
