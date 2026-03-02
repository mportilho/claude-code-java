# API Contract Review Skill

> Audit REST APIs for HTTP semantics, versioning, and consistency

## What It Does

Reviews REST API design for:
- HTTP method semantics and status code correctness
- API versioning strategy
- PATCH behavior and media type clarity
- Request/response structure (DTOs vs entities)
- Error model consistency (prefer Problem Details)
- Backward compatibility and lifecycle signaling

## When to Use

- "Review this API" / "Check REST endpoints"
- "Review this OpenAPI spec"
- Before releasing API changes
- Reviewing controller PRs
- Checking if API follows REST best practices

## Key Concepts

### Audit vs Template

| spring-boot-patterns | api-contract-review |
|---------------------|---------------------|
| How to write controllers | Review existing APIs |
| Templates and examples | Checklist and anti-patterns |
| Creating new code | Auditing existing code |

### Common Issues Caught (Examples)

| Issue | Example |
|-------|---------|
| Wrong method semantics | GET endpoint mutates server state or POST for search instead |
| No versioning | `/users` instead of `/v1/users` |
| Entity leak | JPA entity returned directly |
| 200 with error | `{"status": "error"}` with HTTP 200 |
| Breaking change | Required field added to request |

## Example Usage

```
You: Review the API in UserController

AI Agent: [Checks HTTP verb usage]
          [Validates versioning]
          [Looks for entity leaks]
          [Reviews error handling]
          [Identifies breaking changes]
```

## What It Checks

1. **HTTP Semantics** - Correct verb for operation
2. **URL Design** - Versioning, naming conventions
3. **Request Handling** - Validation, DTOs
4. **Response Design** - DTOs, pagination, consistency
5. **Error Handling** - Status codes, error format
6. **Compatibility** - Breaking vs non-breaking changes

## Related Skills

- `spring-boot-patterns` - Templates for writing controllers (this skill audits them)
- `security-audit` - Security aspects of APIs
- `java-code-review` - General code review (this skill is API-specific)

## References

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 5789 - PATCH Method](https://www.rfc-editor.org/rfc/rfc5789)
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [RFC 9745 - Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745)
- [RFC 8594 - The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)
- [OpenAPI Specification (latest)](https://spec.openapis.org/oas/latest.html)
- [RFC 9110 (HTTP Semantics)](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 5789 (PATCH)](https://www.rfc-editor.org/rfc/rfc5789)
- [RFC 9457 (Problem Details)](https://www.rfc-editor.org/rfc/rfc9457)
- [RFC 9745 (Deprecation Header)](https://www.rfc-editor.org/rfc/rfc9745)
- [RFC 8594 (Sunset Header)](https://www.rfc-editor.org/rfc/rfc8594)
- [OpenAPI latest](https://spec.openapis.org/oas/latest.html)
