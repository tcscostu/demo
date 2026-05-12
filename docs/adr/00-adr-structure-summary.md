# ADR Structure: Audit BFF Architecture Decision Records

## RFC 2119 Requirement Levels
This document and all ADRs follow RFC 2119 standards for requirement levels:

- **MUST** / **REQUIRED** / **SHALL**: Absolute requirement
- **MUST NOT** / **SHALL NOT**: Absolute prohibition
- **SHOULD** / **RECOMMENDED**: Strong recommendation with rare exceptions
- **SHOULD NOT** / **NOT RECOMMENDED**: Strong recommendation against with rare exceptions
- **MAY** / **OPTIONAL**: Truly optional, implementation choice

## ADR Template
Every ADR uses the following structure:

1. **# Issue:** Problem statement and context
2. **# Decision:** Specific solution and implementation details
3. **# Constraints:** Technical/business boundaries
4. **# Alternatives:** Rejected options with brief reasoning
5. **# Rationale:** Justification for the decision
6. **# Implementation Guidelines:** Step-by-step instructions with code/config examples
7. **# Additional Recommendations:** Best practices and future considerations
8. **# Success Criteria:** Completion gate table with validation methods

## ADR Index

| ADR | Title | Scope |
|-----|-------|-------|
| ADR-01 | Runtime and Framework | Java 21, Spring Boot 3.5.3, Kotlin 1.9.25, Maven |
| ADR-02 | API Documentation | SpringDoc OpenAPI, Swagger UI |
| ADR-03 | Package Structure Organization | Simplified hexagonal for BFF |
| ADR-04 | Application Configuration Management | application.yml, upstream URL property |
| ADR-05 | REST API Error Handling | RFC 9457 Problem Details |
| ADR-06 | Structured Logging Strategy | AOP-based request/response logging |
| ADR-07 | Code Quality Standards | Spotless (ktlint), JaCoCo |
| ADR-08 | Upstream Proxy Integration Pattern | RestClient to upstream Audit Event API |
| ADR-09 | Data Transfer Validation | Jakarta Bean Validation on request DTO |
| ADR-10 | Testing Strategy | JUnit 5, MockK, MockMvc, test examples |

## Related Business Requirements
- **BR-002**: Audit BFF Query Proxy
