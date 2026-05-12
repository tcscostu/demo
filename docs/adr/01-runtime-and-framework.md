# Issue: Runtime and Framework Selection for Audit BFF

The Audit BFF requires a standardized runtime and framework selection consistent with the enterprise BFF architecture established by the FSM BFF project (bo-bff-fsm). Without alignment to the existing BFF stack, the team faces fragmented tooling, inconsistent patterns, and increased maintenance burden.

**Impact**: Diverging from the established BFF stack introduces unnecessary learning curves, prevents code reuse, and complicates operational support across BFF services.

# Decision:

Use the following technology stack for the Audit BFF, matching the enterprise BFF standard established by `bo-bff-fsm`:

| Component | Technology | Version |
|-----------|------------|---------|
| **Runtime** | Java (OpenJDK) | 21 LTS |
| **Language** | Kotlin | 1.9.25 |
| **Framework** | Spring Boot | 3.5.3 |
| **Build Tool** | Apache Maven | 3.9.x |
| **Server** | Embedded Tomcat | (via spring-boot-starter-web) |

**Required Components**:
- Spring Boot 3.5.3 with `spring-boot-starter-web` (servlet-based)
- Spring Boot Actuator for health and management endpoints
- Kotlin 1.9.25 with `kotlin-maven-plugin` and Spring/AllOpen compiler plugins
- Jackson Kotlin module for JSON serialization
- JDK 21 LTS with `-Xjsr305=strict` for null-safety interop

# Constraints:

## Technical:
- MUST use JDK 21 or higher (LTS preferred)
- MUST use Kotlin 1.9.25 for consistency with enterprise BFF codebase
- MUST use Spring Boot 3.5.3 parent POM for dependency management
- MUST NOT use Spring WebFlux or reactive stack — servlet-based (spring-boot-starter-web) only
- MUST NOT include `spring-boot-starter-data-jpa` or any database driver — no database dependency
- MUST NOT include `spring-boot-starter-security` — authentication handled by API gateway

## Environmental:
- All team members MUST use identical JDK major versions
- Corporate proxy/firewall configurations may impact Maven dependency resolution
- macOS, Linux, and Windows MUST be supported for development

## Business:
- MUST use LTS Java version for long-term support
- MUST align with existing bo-bff-fsm project conventions to enable knowledge transfer

# Alternatives:

**Quarkus 3.28.3 (platform ADR ecosystem)**: The existing platform ADRs (docs/) target Quarkus + Java. Rejected for the BFF layer because the enterprise BFF standard (bo-bff-fsm) uses Spring Boot + Kotlin, and consistency within the BFF tier is prioritized over alignment with the backend platform tier.

**Spring Boot with Java (no Kotlin)**: Simpler setup without Kotlin compiler plugins. Rejected because the enterprise BFF team has established Kotlin expertise and patterns through bo-bff-fsm.

**Spring Boot 3.4.x**: Older stable version. Rejected because bo-bff-fsm already uses 3.5.3, and version consistency reduces integration issues.

# Rationale:

Spring Boot 3.5.3 with Kotlin 1.9.25 matches the proven enterprise BFF architecture. The team has existing expertise, shared patterns (generic adapters, AOP logging, exception handling), and operational runbooks built around this stack. JDK 21 LTS provides virtual threads and modern language features. Single-module Maven project is appropriate for a simple proxy BFF with no database or complex module separation needs.

# Implementation Guidelines:

## 1. Maven Project Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.3</version>
        <relativePath/>
    </parent>

    <groupId>tr.com.mycorp.audit</groupId>
    <artifactId>bff-audit</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>bff-audit</name>
    <description>BFF project for Enterprise Audit Platform</description>

    <properties>
        <java.version>21</java.version>
        <kotlin.version>1.9.25</kotlin.version>
        <kotlin.code.style>official</kotlin.code.style>
        <opentelemetry.version>1.31.0</opentelemetry.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web (Servlet / Tomcat) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Spring Boot Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Spring Boot AOP -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>

        <!-- Kotlin -->
        <dependency>
            <groupId>com.fasterxml.jackson.module</groupId>
            <artifactId>jackson-module-kotlin</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-reflect</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-stdlib</artifactId>
        </dependency>

        <!-- OpenTelemetry API -->
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-api</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>

        <!-- Testing -->
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
        <dependency>
            <groupId>com.ninja-squad</groupId>
            <artifactId>springmockk</artifactId>
            <version>4.0.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>${project.basedir}/src/main/kotlin</sourceDirectory>
        <testSourceDirectory>${project.basedir}/src/test/kotlin</testSourceDirectory>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <plugin>
                <groupId>org.jetbrains.kotlin</groupId>
                <artifactId>kotlin-maven-plugin</artifactId>
                <configuration>
                    <args>
                        <arg>-Xjsr305=strict</arg>
                    </args>
                    <compilerPlugins>
                        <plugin>spring</plugin>
                        <plugin>all-open</plugin>
                    </compilerPlugins>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.jetbrains.kotlin</groupId>
                        <artifactId>kotlin-maven-allopen</artifactId>
                        <version>${kotlin.version}</version>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>
</project>
```

**Key differences from bo-bff-fsm:**
- No `spring-boot-starter-data-jpa` (no database)
- No `ojdbc11` database driver
- No JPA/noarg compiler plugin (no entities)
- Added `spring-boot-starter-validation` for request DTO validation

## 2. Application Entry Point

```kotlin
package tr.com.mycorp.audit.bff

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class BffAuditApplication

fun main(args: Array<String>) {
    runApplication<BffAuditApplication>(*args)
}
```

## 3. JDK Setup

```bash
# Using SDKMAN (recommended)
sdk install java 21.0.4-tem
sdk use java 21.0.4-tem

# Verify
java -version   # openjdk version "21.0.4"
javac -version   # javac 21.0.4
```

# Additional Recommendations:

- **Version Pinning**: Consider adding `.sdkmanrc` with `java=21.0.4-tem` for team consistency
- **Container Images**: When containerizing, use `eclipse-temurin:21-jre-alpine` for minimal runtime images
- **IDE**: Configure IntelliJ IDEA with Kotlin plugin and JDK 21 as project SDK
- **Future**: When Spring Boot 3.6.x is released and stable in bo-bff-fsm, upgrade both projects in tandem

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| JDK Version | `java -version 2>&1 \| head -1` | Output contains `21` |
| Kotlin Version | `grep kotlin.version pom.xml` | `1.9.25` |
| Spring Boot Version | `grep spring-boot-starter-parent pom.xml` | `3.5.3` |
| No Database Dependency | `grep -c "data-jpa\|ojdbc\|datasource" pom.xml` | `0` |
| No Security Dependency | `grep -c "spring-boot-starter-security" pom.xml` | `0` |
| Build Success | `./mvnw clean compile` | Exit code 0 |
| Application Starts | `./mvnw spring-boot:run` | Starts without errors on configured port |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Runtime and Framework..."

# Check Java version
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
if [[ "$JAVA_VERSION" != 21.* ]]; then
  echo "❌ Java version $JAVA_VERSION detected. Required: 21.x"
  exit 1
fi
echo "✅ Java version: $JAVA_VERSION"

# Check Kotlin version in pom.xml
if grep -q "<kotlin.version>1.9.25</kotlin.version>" pom.xml; then
  echo "✅ Kotlin version: 1.9.25"
else
  echo "❌ Kotlin version mismatch"
  exit 1
fi

# Check Spring Boot version
if grep -q "<version>3.5.3</version>" pom.xml; then
  echo "✅ Spring Boot version: 3.5.3"
else
  echo "❌ Spring Boot version mismatch"
  exit 1
fi

# Verify no database dependencies
if grep -qE "data-jpa|ojdbc|datasource" pom.xml; then
  echo "❌ Database dependency found — must be removed"
  exit 1
fi
echo "✅ No database dependencies"

# Build
./mvnw clean compile -q
echo "✅ Project compiles successfully"

echo "✅ All Runtime and Framework criteria met"
```
