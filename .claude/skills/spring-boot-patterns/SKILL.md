---
name: spring-boot-patterns
description: Spring Boot 4.x / Spring Framework 7.x best practices and patterns (Java 25+). Use when creating controllers, services, repositories, or when user asks about Spring Boot architecture, REST APIs, exception handling, or JPA patterns.
---

# Spring Boot Patterns Skill

Best practices and patterns for modern Spring Boot applications (Spring Boot 4.x, Java 25+ with JDK 21 compatibility).

## When to Use
- User says "create controller" / "add service" / "Spring Boot help"
- Reviewing Spring Boot code
- Setting up new Spring Boot project structure

## Project Structure

```
src/main/java/com/example/myapp/
├── MyAppApplication.java          # @SpringBootApplication
├── config/                        # Configuration classes
│   ├── SecurityConfig.java
│   └── WebConfig.java
├── controller/                    # REST controllers
│   └── UserController.java
├── service/                       # Business logic
│   ├── UserService.java
│   └── impl/
│       └── UserServiceImpl.java
├── repository/                    # Data access
│   └── UserRepository.java
├── model/                         # Entities
│   └── User.java
├── dto/                           # Data transfer objects
│   ├── request/
│   │   └── CreateUserRequest.java
│   └── response/
│       └── UserResponse.java
├── exception/                     # Custom exceptions
│   ├── ResourceNotFoundException.java
│   └── GlobalExceptionHandler.java
└── util/                          # Utilities
    └── DateUtils.java
```

---

## Controller Patterns

### REST Controller Template
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public ResponseEntity<List<UserResponse>> getAll() {
        return ResponseEntity.ok(userService.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Controller Best Practices

| Practice | Example |
|----------|---------|
| Versioned API | `/api/v1/users` |
| Plural nouns | `/users` not `/user` |
| HTTP methods | GET=read, POST=create, PUT=update, DELETE=delete |
| Status codes | 200=OK, 201=Created, 204=NoContent, 404=NotFound |
| Validation | `@Valid` on request body |

### ❌ Anti-patterns
```java
// ❌ Business logic in controller
@PostMapping
public User create(@RequestBody User user) {
    user.setCreatedAt(LocalDateTime.now());  // Logic belongs in service
    return userRepository.save(user);         // Direct repo access
}

// ❌ Returning entity directly (exposes internals)
@GetMapping("/{id}")
public User getById(@PathVariable Long id) {
    return userRepository.findById(id).get();
}
```

---

## HTTP Client Patterns

### RestClient (Modern alternative to RestTemplate)
```java
@Service
public class ExternalApiClient {
    
    // Auto-configured by Spring Boot 
    private final RestClient restClient;
    
    public ExternalApiClient(RestClient.Builder restClientBuilder) {
        this.restClient = restClientBuilder.baseUrl("https://api.example.com").build();
    }
    
    public ExternalData getData(String id) {
        return restClient.get()
            .uri("/data/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new BusinessException("CLIENT_ERROR", "External API error");
            })
            .body(ExternalData.class);
    }
}
```

---

## Service Patterns

### Service Interface + Implementation
```java
// Interface
public interface UserService {
    List<UserResponse> findAll();
    UserResponse findById(Long id);
    UserResponse create(CreateUserRequest request);
    UserResponse update(Long id, UpdateUserRequest request);
    void delete(Long id);
}

// Implementation
@Service
@Transactional(readOnly = true)  // Default read-only
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    public UserServiceImpl(UserRepository userRepository, UserMapper userMapper) {
        this.userRepository = userRepository;
        this.userMapper = userMapper;
    }

    @Override
    public List<UserResponse> findAll() {
        return userRepository.findAll().stream()
            .map(userMapper::toResponse)
            .toList();
    }

    @Override
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    @Override
    @Transactional  // Write transaction
    public UserResponse create(CreateUserRequest request) {
        User user = userMapper.toEntity(request);
        User saved = userRepository.save(user);
        return userMapper.toResponse(saved);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", id);
        }
        userRepository.deleteById(id);
    }
}
```

### Service Best Practices

- Interface + Impl for testability
- `@Transactional(readOnly = true)` at class level
- `@Transactional` for write methods
- Throw domain exceptions, not generic ones
- Use mappers (MapStruct) for entity ↔ DTO conversion

---

## Repository Patterns

### JPA Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query
    Optional<User> findByEmail(String email);

    List<User> findByActiveTrue();

    // Custom query
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId")
    List<User> findByDepartmentId(@Param("deptId") Long departmentId);

    // Native query (use sparingly)
    @Query(value = "SELECT * FROM users WHERE created_at > :date",
           nativeQuery = true)
    List<User> findRecentUsers(@Param("date") LocalDate date);

    // Exists check (more efficient than findBy)
    boolean existsByEmail(String email);

    // Count
    long countByActiveTrue();
}
```

### JdbcClient (Modern fluent JDBC API)
When JPA is overkill or you need complex native queries, use `JdbcClient` over `JdbcTemplate`:

```java
@Repository
public class UserJdbcRepository {

    private final JdbcClient jdbcClient;

    public UserJdbcRepository(JdbcClient jdbcClient) {
        this.jdbcClient = jdbcClient;
    }

    public Optional<User> findByEmail(String email) {
        return jdbcClient.sql("SELECT * FROM users WHERE email = :email")
            .param("email", email)
            .query(User.class)
            .optional();
    }
}
```

### Repository Best Practices

- Use derived queries when possible
- `Optional` for single results
- `existsBy` instead of `findBy` for existence checks
- Avoid native queries unless necessary
- Use `@EntityGraph` for fetch optimization

---

## DTO Patterns

### Request/Response DTOs
```java
// Request DTO with validation
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,

    @NotBlank
    @Email(message = "Invalid email format")
    String email,

    @NotNull
    @Min(18)
    Integer age
) {}

// Response DTO
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}
```

### MapStruct Mapper
```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    UserResponse toResponse(User entity);

    List<UserResponse> toResponseList(List<User> entities);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    User toEntity(CreateUserRequest request);
}
```

---

## Exception Handling

### Custom Exceptions
```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String resource, Long id) {
        super(String.format("%s not found with id: %d", resource, id));
    }
}

public class BusinessException extends RuntimeException {

    private final String code;

    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
}
```

### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setProperty("code", "NOT_FOUND");
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation Error");
        problem.setProperty("code", "VALIDATION_ERROR");
        problem.setProperty("errors", errors);
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleException(Exception ex) {
        log.error("Unexpected error", ex);
        
        // Java 21+ Pattern Matching for switch
        return switch (ex) {
            case IllegalArgumentException e -> {
                ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, e.getMessage());
                pd.setTitle("Bad Request");
                pd.setProperty("code", "BAD_REQUEST");
                yield pd;
            }
            case AccessDeniedException e -> {
                ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.FORBIDDEN, "Access Denied");
                pd.setTitle("Forbidden");
                pd.setProperty("code", "FORBIDDEN");
                yield pd;
            }
            default -> {
                ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
                pd.setTitle("Internal Server Error");
                pd.setProperty("code", "INTERNAL_ERROR");
                yield pd;
            }
        };
    }
}

// Spring Boot native ProblemDetail (RFC 7807) replaces custom ErrorResponse classes
```

---

## Configuration Patterns

### Application Properties
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true     # Enable Java Virtual Threads (Project Loom)
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # Never 'create' in production!
    show-sql: false

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000
```

### Configuration Properties Class
```java
@Configuration
@ConfigurationProperties(prefix = "app.jwt")
@Validated
public class JwtProperties {

    @NotBlank
    private String secret;

    @Min(60000)
    private long expiration;

    // getters and setters
}
```

### Profile-Specific Configuration
```
src/main/resources/
├── application.yml           # Common config
├── application-dev.yml       # Development
├── application-test.yml      # Testing
└── application-prod.yml      # Production
```

---

## Common Annotations Quick Reference

| Annotation | Purpose |
|------------|---------|
| `@RestController` | REST controller (combines @Controller + @ResponseBody) |
| `@Service` | Business logic component |
| `@Repository` | Data access component |
| `@Configuration` | Configuration class |
| `@Transactional` | Transaction management |
| `@Valid` | Trigger validation |
| `@ConfigurationProperties` | Bind properties to class |
| @Profile("dev") | Profile-specific bean |
| @Scheduled | Scheduled tasks |

---

## Modern Concurrency (Java 21+ / 25+)

When dealing with multiple independent subtasks within a service layer, rely on **Structured Concurrency** (`StructuredTaskScope`) alongside virtual threads. In Java 25+, you should also use **Scoped Values** (`ScopedValue<>`) instead of `ThreadLocal` for safe, efficient context propagation.

```java
@Service
public class ReportService {

    // Define a ScopedValue for context sharing (e.g., Tenant ID or Security Context)
    public static final ScopedValue<String> TENANT_ID = ScopedValue.newInstance();

    public CombinedReport generateReport(String tenantId) throws InterruptedException {
        // Bind the scoped value
        return ScopedValue.where(TENANT_ID, tenantId).get(() -> {
            try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                // Fork independent tasks using virtual threads. 
                // Scoped values are automatically inherited by the child threads in the scope.
                StructuredTaskScope.Subtask<UserData> userSubtask = scope.fork(this::fetchUserData);
                StructuredTaskScope.Subtask<SalesData> salesSubtask = scope.fork(this::fetchSalesData);

                scope.join();           // Join all subtasks
                scope.throwIfFailed();  // Propagate exceptions if any subtask failed

                // Aggregate results safely
                return new CombinedReport(userSubtask.get(), salesSubtask.get());
            } catch (ExecutionException e) {
                throw new RuntimeException("Failed to generate report", e);
            }
        });
    }
    
    private UserData fetchUserData() {
        // Access the scoped value efficiently without ThreadLocal overhead
        String currentTenant = TENANT_ID.get(); 
        // ... fetch data for tenant
        return new UserData();
    }

    private SalesData fetchSalesData() {
        String currentTenant = TENANT_ID.get();
        // ... fetch data for tenant
        return new SalesData();
    }
}
```

---

## Testing Patterns

### Controller Test (MockMvc)
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L))
            .thenReturn(new UserResponse(1L, "John", "john@example.com", null));

        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

### Service Test
```java
@ExtendWith(MockitoExtension.class)
class UserServiceImplTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    void shouldThrowWhenUserNotFound() {
        when(userRepository.findById(1L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(1L))
            .isInstanceOf(ResourceNotFoundException.class);
    }
}
```

### Integration Test
```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class UserIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17");

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateUser() throws Exception {
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "John", "email": "john@example.com", "age": 25}
                    """))
            .andExpect(status().isCreated());
    }
}
```

---

## Quick Reference Card

| Layer | Responsibility | Annotations |
|-------|---------------|-------------|
| Controller | HTTP handling, validation | `@RestController`, `@Valid` |
| Service | Business logic, transactions | `@Service`, `@Transactional` |
| Repository | Data access | `@Repository`, extends `JpaRepository` |
| DTO | Data transfer | Records with validation annotations |
| Config | Configuration | `@Configuration`, `@ConfigurationProperties` |
| Exception | Error handling | `@RestControllerAdvice` |
