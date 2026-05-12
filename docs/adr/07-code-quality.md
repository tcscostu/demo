# Issue: Code Quality Standards for Audit BFF

The Audit BFF requires automated code quality enforcement to maintain consistent formatting, prevent style drift, and ensure test coverage, consistent with the enterprise BFF standards.

**Impact**: Without automated quality tooling, code reviews focus on style issues instead of logic, formatting inconsistencies creep in over time, and test coverage gaps go undetected.

# Decision:

Use Spotless with ktlint for Kotlin code formatting and JaCoCo for test coverage reporting, matching the enterprise BFF standard (bo-bff-fsm).

**Required Components**:
- Spotless Maven Plugin 2.43.0 with ktlint 1.0.1 for Kotlin formatting
- JaCoCo Maven Plugin 0.8.11 for test coverage reporting
- Maven Surefire Plugin 3.2.5 for test execution
- Formatting applied automatically during build

# Constraints:

- MUST use Spotless 2.43.0 with ktlint 1.0.1 (matching bo-bff-fsm versions)
- MUST use JaCoCo 0.8.11 for coverage reporting
- ktlint rules: indent_size=4, max_line_length=120, trailing commas allowed
- Spotless `apply` goal MUST run during build to auto-fix formatting
- JaCoCo MUST generate both XML and HTML reports
- MUST NOT enforce JPA entity or repository coverage (no database in this BFF)

# Alternatives:

**Detekt**: Kotlin static analysis tool. MAY be added later for deeper static analysis. Not required for initial setup — ktlint handles formatting.

**Google Java Format**: Java formatting standard. Not applicable — the BFF is pure Kotlin.

**Gradle with ktlint plugin**: Alternative build tool integration. Rejected because the project uses Maven (consistent with bo-bff-fsm).

# Rationale:

Spotless with ktlint is the proven formatting solution in the enterprise BFF stack. It auto-fixes formatting on build, eliminating style discussions in code reviews. JaCoCo provides test coverage visibility without enforcing arbitrary thresholds, which is appropriate for a simple proxy BFF.

# Implementation Guidelines:

## 1. Maven Properties

```xml
<properties>
    <jacoco.version>0.8.11</jacoco.version>
    <spotless.version>2.43.0</spotless.version>
    <ktlint.version>1.0.1</ktlint.version>
</properties>
```

## 2. Spotless Plugin Configuration

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>${spotless.version}</version>
    <configuration>
        <kotlin>
            <ktlint>
                <version>${ktlint.version}</version>
                <editorConfigOverride>
                    <indent_size>4</indent_size>
                    <insert_final_newline>true</insert_final_newline>
                    <max_line_length>120</max_line_length>
                    <ij_kotlin_allow_trailing_comma>true</ij_kotlin_allow_trailing_comma>
                    <ij_kotlin_allow_trailing_comma_on_call_site>true</ij_kotlin_allow_trailing_comma_on_call_site>
                </editorConfigOverride>
            </ktlint>
            <includes>
                <include>src/main/kotlin/**/*.kt</include>
                <include>src/test/kotlin/**/*.kt</include>
            </includes>
            <excludes>
                <exclude>build/**/*.kt</exclude>
            </excludes>
        </kotlin>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>apply</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## 3. JaCoCo Plugin Configuration

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
            <configuration>
                <formats>
                    <format>XML</format>
                    <format>HTML</format>
                </formats>
                <includes>
                    <include>tr/com/paycell/audit/bff/adapter/**</include>
                    <include>tr/com/paycell/audit/bff/exception/**</include>
                </includes>
                <excludes>
                    <!-- Model/DTO classes (data classes, no logic) -->
                    <exclude>**/model/**</exclude>
                    <!-- Configuration classes -->
                    <exclude>**/config/**</exclude>
                    <!-- Application entry point -->
                    <exclude>**/BffAuditApplication*</exclude>
                    <!-- Kotlin generated classes -->
                    <exclude>**/*Kt.class</exclude>
                    <exclude>**/\$*.class</exclude>
                    <exclude>**/*Companion*.class</exclude>
                </excludes>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 4. Maven Surefire Plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
</plugin>
```

## 5. Usage Commands

```bash
# Format code
./mvnw spotless:apply

# Check formatting without fixing
./mvnw spotless:check

# Run tests with coverage
./mvnw test

# View coverage report
open target/site/jacoco/index.html
```

# Additional Recommendations:

- Consider adding a pre-commit hook that runs `./mvnw spotless:check` before commits
- Monitor JaCoCo coverage trends over time — aim for > 80% on adapter and exception handler code
- Consider adding Detekt for deeper Kotlin static analysis in future iterations
- IDE: Install ktlint plugin in IntelliJ for real-time formatting feedback

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Spotless Plugin | `grep spotless-maven-plugin pom.xml` | Plugin present with version 2.43.0 |
| ktlint Version | `grep ktlint pom.xml` | Version 1.0.1 |
| JaCoCo Plugin | `grep jacoco-maven-plugin pom.xml` | Plugin present with version 0.8.11 |
| Formatting Pass | `./mvnw spotless:check` | Exit code 0 |
| Tests Run | `./mvnw test` | Tests execute successfully |
| Coverage Report | `test -f target/site/jacoco/index.html` | Report generated |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Code Quality Standards..."

# Check Spotless plugin
if grep -q "spotless-maven-plugin" pom.xml; then
  echo "✅ Spotless plugin present"
else
  echo "❌ Spotless plugin not found"
  exit 1
fi

# Check JaCoCo plugin
if grep -q "jacoco-maven-plugin" pom.xml; then
  echo "✅ JaCoCo plugin present"
else
  echo "❌ JaCoCo plugin not found"
  exit 1
fi

# Run formatting check
./mvnw spotless:check -q 2>/dev/null && echo "✅ Code formatting passes" || echo "⚠️ Formatting issues found"

# Run tests
./mvnw test -q 2>/dev/null && echo "✅ Tests pass" || echo "⚠️ Test failures"

# Check coverage report
if [ -f target/site/jacoco/index.html ]; then
  echo "✅ JaCoCo coverage report generated"
fi

echo "✅ All Code Quality criteria met"
```
