---
name: api-contract-review
description: Review REST API contracts for HTTP semantics, versioning, backward compatibility, and response consistency. Use when user asks "review API", "check endpoints", "REST review", or before releasing API changes.
---

# API Contract Review Skill

Audit REST API design for correctness, consistency, and compatibility.

## When to Use
- User asks "review this API", "check REST endpoints", "contract review"
- Before releasing API/controller changes
- Reviewing PRs that change routes, payloads, status codes, or OpenAPI specs
- Checking backward compatibility across versions

---

## Standards Baseline (Use as Source of Truth)

- HTTP semantics and status codes: RFC 9110 (June 2022)
- PATCH behavior: RFC 5789 (March 2010)
- Problem Details error format: RFC 9457 (July 2023, obsoletes RFC 7807)
- API contract schema language: OpenAPI Specification (latest published version)

If project conventions conflict with RFC/OpenAPI semantics, flag as **High** and explain with reference section.

---

## Quick Reference: High-Value Findings

| Severity | Issue | Symptom | Impact |
|----------|-------|---------|--------|
| High | Wrong method semantics | Unsafe action on GET | Side effects, crawler/caching risks |
| High | Missing versioning | `/users` instead of `/v1/users` | Breaking changes affect all clients |
| High | Incorrect status semantics | 200 carrying business error | Client retry/error handling breaks |
| High | Breaking contract change | Removed/renamed response field | Existing clients fail |
| High | Entity leak | JPA entity in response | Exposes internals, N+1 risk |
| Medium | Unclear PATCH media type | PATCH without defined patch format | Interop failures |
| Medium | Missing lifecycle signaling | Deprecated endpoint with no timeline | Migration risk |
| Low | Naming inconsistency | `/getUsers` vs `/users` | Learnability/consistency issues |

---

## Review Workflow (Evidence-First)

### 1. Map Contract Surface
- Identify all public endpoints and operation signatures (method + path + request + responses)
- Identify declared API contract source (OpenAPI file and/or annotations)

### 2. Validate HTTP Semantics
- Evaluate method safety/idempotency against RFC 9110
- Validate status code semantics against actual behavior
- For PATCH, verify media type and patch semantics are explicit (RFC 5789)

### 3. Validate Contract Shape
- Ensure request/response schemas are explicit (DTO/Schema, not persistence entities)
- Ensure error payload is standardized (prefer RFC 9457 Problem Details)
- Check pagination/filter contracts for collection endpoints

### 4. Validate Compatibility and Lifecycle
- Classify changes as breaking vs non-breaking
- Check deprecation communication (headers/docs/timeline/migration)

### 5. Verb Selection Guide

| Verb | Use For | Idempotent | Safe | Request Body |
|------|---------|------------|------|--------------|
| GET | Retrieve resource | Yes | Yes | No |
| POST | Create new resource | No | No | Yes |
| PUT | Replace entire resource | Yes | No | Yes |
| PATCH | Partial update | No* | No | Yes |
| DELETE | Remove resource | Yes | No | Optional |

*PATCH can be idempotent depending on implementation

---

## HTTP Method and Body Semantics

```java
// ✅ GET for retrieval (safe, idempotent)
@GetMapping("/users")
public List<UserResponse> listUsers(
    @RequestParam String name,
    @RequestParam(required = false) String email) { }

// ❌ GET causing state change
@GetMapping("/users/{id}/activate")
public void activateUser(@PathVariable Long id) { }

// ✅ Use POST/PATCH/PUT depending on operation semantics
@PostMapping("/users/{id}/activate")
public ResponseEntity<Void> activateUser(@PathVariable Long id) { }

// ❌ POST for idempotent update
@PostMapping("/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody UserDto dto) { }

// ✅ PUT full replacement (idempotent)
@PutMapping("/users/{id}")
public UserResponse replaceUser(@PathVariable Long id, @RequestBody UserReplaceRequest dto) { }

// ✅ PATCH partial modification
@PatchMapping("/users/{id}")
public UserResponse patchUser(@PathVariable Long id, @RequestBody UserPatchRequest dto) { }
```

---

## API Versioning

### Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/v1/users` | Clear, easy routing | URL changes |
| Header | `Accept: application/vnd.api.v1+json` | Clean URLs | Hidden, harder to test |
| Query param | `/users?version=1` | Easy to add | Easy to forget |

### Recommended: URL Path

```java
// ✅ Versioned endpoints
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { }

// ❌ No versioning
@RestController
@RequestMapping("/api/users")  // Breaking changes affect everyone
public class UserController { }
```

### Version Checklist
- [ ] All public APIs have version in path
- [ ] Internal APIs documented as internal (or versioned too)
- [ ] Deprecation strategy defined for old versions

---

## Request/Response Design

### DTO vs Entity

```java
// ❌ Entity in response (leaks internals)
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
    // Exposes: password hash, internal IDs, lazy collections
}

// ✅ DTO response
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    return UserResponse.from(user);  // Only public fields
}
```

### Response Consistency

```java
// ❌ Inconsistent responses
@GetMapping("/users")
public List<User> getUsers() { }  // Returns array

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { }  // Returns object

@GetMapping("/users/count")
public int countUsers() { }  // Returns primitive

// ✅ Consistent wrapper (optional but recommended for large APIs)
@GetMapping("/users")
public ApiResponse<List<UserResponse>> getUsers() {
    return ApiResponse.success(userService.findAll());
}

// Or at minimum, consistent structure:
// - Collections: always wrapped or always raw (pick one)
// - Single items: always object
// - Counts/stats: always object { "count": 42 }
```

### Pagination

```java
// ❌ No pagination on collections
@GetMapping("/users")
public List<User> getAllUsers() {
    return userRepository.findAll();  // Could be millions
}

// ✅ Paginated
@GetMapping("/users")
public Page<UserResponse> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {
    return userService.findAll(PageRequest.of(page, size));
}
```

---

## HTTP Status Codes

| Code | Typical Contract Meaning |
|------|--------------------------|
| 200 OK | Successful retrieval/update response with body |
| 201 Created | New resource created; use `Location` for primary resource URI |
| 202 Accepted | Accepted for async processing |
| 204 No Content | Success with no response content |
| 400 Bad Request | Invalid syntax/invalid request framing |
| 401 Unauthorized | Missing/invalid authentication credentials |
| 403 Forbidden | Authenticated but not permitted |
| 404 Not Found | Target resource not found |
| 409 Conflict | Request conflicts with current resource state (Duplicate, concurrent modification) |
| 412 Precondition Failed | Conditional request precondition failed |
| 415 Unsupported Media Type | Request media type unsupported |
| 422 Unprocessable Content | Syntactically valid content but semantically unprocessable |
| 500 Internal Server Error | Unexpected server failure |

### Anti-Pattern: Success Code for Failure

```java
// ❌ NEVER DO THIS: Returns 200 for domain failure
@GetMapping("/{id}")
public ResponseEntity<Map<String, Object>> getUser(@PathVariable Long id) {
    try {
        User user = userService.findById(id);
        return ResponseEntity.ok(Map.of("status", "success", "data", user));
    } catch (NotFoundException e) {
        return ResponseEntity.ok(Map.of(  // Still 200!
            "status", "error",
            "message", "User not found"
        ));
    }
}

// ✅ Status code matches outcome
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

---

## Error Contract (Prefer RFC 9457 Problem Details)

Preferred payload members:
- `type` (problem type URI)
- `title` (short summary)
- `status` (HTTP status code; advisory but must match response status)
- `detail` (human-readable detail)
- `instance` (resource/problem instance URI)

Spring example:

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setType(URI.create("https://example.com/problems/resource-not-found"));
    pd.setTitle("Resource not found");
    pd.setDetail(ex.getMessage());
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(pd);
}
```

### Security: Don't Expose Internals

```java
// ❌ Exposes stack trace
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleAll(Exception ex) {
    return ResponseEntity.status(500)
        .body(ex.getStackTrace().toString());  // Security risk!
}

// ✅ Generic message, log details server-side
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleAll(Exception ex) {
    log.error("Unexpected error", ex);  // Full details in logs
    return ResponseEntity.status(500)
        .body(ErrorResponse.of("INTERNAL_ERROR", "An unexpected error occurred"));
}
```

---

## Backward Compatibility

### Breaking Changes (Avoid in Same Version)

| Change | Breaking? | Migration |
|--------|-----------|-----------|
| Remove endpoint | Yes | Deprecate first, remove in next version |
| Remove field from response | Yes | Keep field, return null/default |
| Add required field to request | Yes | Make optional with default |
| Change field type | Yes | Add new field, deprecate old |
| Rename field | Yes | Support both temporarily |
| Change URL path | Yes | Redirect old to new |

### Usually Non-Breaking

- Add optional request field
- Add new response field (if clients tolerate unknown fields)
- Add new endpoint
- Add new optional query parameter

### Deprecation Pattern

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {

    @Deprecated
    @GetMapping("/by-email")  // Old endpoint
    public UserResponse getByEmailOld(@RequestParam String email) {
        return getByEmail(email);  // Delegate to new
    }

    @GetMapping(params = "email")  // New pattern
    public UserResponse getByEmail(@RequestParam String email) {
        return userService.findByEmail(email);
    }
}
```

---

## API Review Checklist

### 1. HTTP Semantics
- [ ] GET for retrieval only (no side effects)
- [ ] POST for creation (returns 201 + Location)
- [ ] PUT for full replacement (idempotent)
- [ ] PATCH for partial updates
- [ ] DELETE for removal (idempotent)

### 2. URL Design
- [ ] Versioned (`/v1/`, `/v2/`)
- [ ] Nouns, not verbs (`/users`, not `/getUsers`)
- [ ] Plural for collections (`/users`, not `/user`)
- [ ] Hierarchical for relationships (`/users/{id}/orders`)
- [ ] Consistent naming, using kebab-case

### 3. Request Handling
- [ ] Request and response schemas are explicit (DTO/schema-first)
- [ ] Validation with `@Valid`
- [ ] Clear error messages for validation failures
- [ ] No persistence entities leaked directly in public API
- [ ] Reasonable size limits
- [ ] Collection endpoints have bounded/paginated contract

### 4. Response Design
- [ ] Response DTOs (not entities)
- [ ] Consistent structure across endpoints
- [ ] Pagination for collections
- [ ] Proper status codes (not 200 for errors)
- [ ] `201` responses include resource location when applicable
- [ ] `204` responses return no content
- [ ] Async flows use `202` where appropriate

### 5. Error Handling
- [ ] Error payload format is consistent (prefer RFC 9457)
- [ ] Error payload does not contradict HTTP status
- [ ] Machine-readable error codes
- [ ] Human-readable messages
- [ ] No stack traces exposed
- [ ] Proper 4xx vs 5xx distinction

### 6. Compatibility
- [ ] No breaking changes in current version
- [ ] Deprecated endpoints documented
- [ ] Migration path for breaking changes

---

## Token Optimization

For large APIs:
1. List all controllers: `find . -name "*Controller.java"`
2. Sample 2-3 controllers for pattern analysis
3. Check `@ExceptionHandler` configuration once
4. Grep for specific anti-patterns:
   ```bash
   # Find potential entity leaks
   grep -r "public.*Entity.*@GetMapping" --include="*.java"

   # Find 200 with error patterns
   grep -r "ResponseEntity.ok.*error" --include="*.java"

   # Find unversioned APIs
   grep -r "@RequestMapping.*api" --include="*.java" | grep -v "/v[0-9]"
   ```

---

## References

- RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- RFC 5789: https://www.rfc-editor.org/rfc/rfc5789
- RFC 9457: https://www.rfc-editor.org/rfc/rfc9457
- RFC 9745: https://www.rfc-editor.org/rfc/rfc9745
- RFC 8594: https://www.rfc-editor.org/rfc/rfc8594
- OpenAPI latest: https://spec.openapis.org/oas/latest.html