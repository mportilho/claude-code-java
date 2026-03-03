---
name: logging-patterns
description: Java logging best practices with SLF4J 2.0+, structured logging (JSON), and context propagation (Micrometer) for request tracing, including virtual threads. Includes AI-friendly log formats for Claude Code debugging. Use when user asks about logging, debugging application flow, or analyzing logs.
---

# Logging Patterns Skill

Effective logging for Java applications with focus on structured, AI-parsable formats.

## When to Use

- User says "add logging" / "improve logs" / "debug this"
- Analyzing application flow from logs
- Setting up structured logging (JSON)
- Request tracing with correlation IDs
- AI/Claude Code needs to analyze application behavior

---

## AI-Friendly Logging

> **Key insight:** JSON logs are better for AI analysis - faster parsing, fewer tokens, direct field access.

### Why JSON for AI/Claude Code?

```
# Text format - AI must "interpret" the string
2026-01-29 10:15:30 INFO OrderService - Order 12345 created for user-789, total: 99.99

# JSON format - AI extracts fields directly
{"timestamp":"2026-01-29T10:15:30Z","level":"INFO","orderId":12345,"userId":"user-789","total":99.99}
```

| Aspect           | Text                       | JSON                |
| ---------------- | -------------------------- | ------------------- |
| Parsing          | Regex/interpretation       | Direct field access |
| Token usage      | Higher (repeated patterns) | Lower (structured)  |
| Error extraction | Parse stack trace text     | `exception` field   |
| Filtering        | grep patterns              | `jq` queries        |

### Recommended Setup for AI-Assisted Development

```yaml
# application.yml - JSON by default
logging:
  structured:
    format:
      console: logstash # Spring Boot 3.4+


# When YOU need to read logs manually:
# Option 1: Use jq
# tail -f app.log | jq .

# Option 2: Switch profile temporarily
# java -jar app.jar --spring.profiles.active=human-logs
```

### Log Format Optimized for AI Analysis

```json
{
  "timestamp": "2026-01-29T10:15:30.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created",
  "requestId": "req-abc123",
  "traceId": "trace-xyz",
  "orderId": 12345,
  "userId": "user-789",
  "duration_ms": 45,
  "step": "payment_completed"
}
```

**Key fields for AI debugging:**

- `requestId` - group all logs from same request
- `step` - track progress through flow
- `duration_ms` - identify slow operations
- `level` - quick filter for errors

### Reading Logs with AI/Claude Code

When asking AI to analyze logs:

```bash
# Get recent errors
cat app.log | jq 'select(.level == "ERROR")' | tail -20

# Follow specific request
cat app.log | jq 'select(.requestId == "req-abc123")'

# Find slow operations
cat app.log | jq 'select(.duration_ms > 1000)'
```

AI can then:

1. Parse JSON directly (no guessing)
2. Follow request flow via requestId
3. Identify exactly where errors occurred
4. Measure timing between steps

---

## Quick Setup (Spring Boot 3.4+ / 4.x)

### Native Structured Logging

Spring Boot 3.4+ and 4.x have built-in support - no extra dependencies!

```yaml
# application.yml
logging:
  structured:
    format:
      console: logstash # or "ecs" for Elastic Common Schema


# Supported formats: logstash, ecs, gelf
```

### Profile-Based Switching

```yaml
# application.yml (default - JSON for AI/prod)
spring:
  profiles:
    default: json-logs

---
spring:
  config:
    activate:
      on-profile: json-logs
logging:
  structured:
    format:
      console: logstash

---
spring:
  config:
    activate:
      on-profile: human-logs
# No structured format = human-readable default
logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"
```

**Usage:**

```bash
# Default: JSON (for AI, CI/CD, production)
./mvnw spring-boot:run

# Human-readable when needed
./mvnw spring-boot:run -Dspring.profiles.active=human-logs
```

---

## Setup for Spring Boot < 3.4

### Logstash Logback Encoder

**pom.xml:**

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

**logback-spring.xml:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- JSON (default) -->
    <springProfile name="!human-logs">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>requestId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

    <!-- Human-readable (optional) -->
    <springProfile name="human-logs">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

### Adding Custom Fields

**Modern Java (SLF4J 2.0+ Fluent API):**
_No extra dependencies required. Standard with Spring Boot 3+ / 4.x_

```java
// Fields appear as separate JSON keys in structured logs
log.atInfo()
   .setMessage("Order created")
   .addKeyValue("orderId", order.getId())
   .addKeyValue("userId", user.getId())
   .addKeyValue("total", order.getTotal())
   .addKeyValue("step", "order_created")
   .log();

// Output:
// {"message":"Order created","orderId":123,"userId":"u-456","total":99.99,"step":"order_created"}
```

**Legacy (Logstash Encoder pre-SLF4J 2.0):**

```java
import static net.logstash.logback.argument.StructuredArguments.kv;

log.info("Order created",
    kv("orderId", order.getId()),
    kv("userId", user.getId()),
    kv("total", order.getTotal()),
    kv("step", "order_created")
);
```

---

## SLF4J Basics

### Logger Declaration

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
}

// Or with Lombok
@Slf4j
@Service
public class OrderService {
    // use `log` directly
}
```

### Parameterized Logging

```java
// ✅ GOOD: Evaluated only if level enabled
log.debug("Processing order {} for user {}", orderId, userId);

// ❌ BAD: Always concatenates
log.debug("Processing order " + orderId + " for user " + userId);

// ✅ For expensive operations
log.atDebug().log(() -> "Full order details: " + order.toJson());
```

---

## Log Levels

| Level     | When                       | Example                              |
| --------- | -------------------------- | ------------------------------------ |
| **ERROR** | Failures needing attention | Unhandled exception, service down    |
| **WARN**  | Unexpected but handled     | Retry succeeded, deprecated API used |
| **INFO**  | Business events            | Order created, payment processed     |
| **DEBUG** | Technical details          | Method params, SQL queries           |
| **TRACE** | Very detailed              | Loop iterations (rarely used)        |

```java
log.atError().setMessage("Payment failed").addKeyValue("orderId", id).addKeyValue("reason", reason).setCause(exception).log();
log.atWarn().setMessage("Retry succeeded").addKeyValue("attempt", 3).addKeyValue("orderId", id).log();
log.atInfo().setMessage("Order shipped").addKeyValue("orderId", id).addKeyValue("trackingNumber", tracking).log();
log.atDebug().setMessage("Fetching from DB").addKeyValue("query", "findById").addKeyValue("id", id).log();
```

---

## MDC (Mapped Diagnostic Context)

MDC adds context to every log entry in a request - essential for tracing.

### Request ID Filter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestContextFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        try {
            String requestId = Optional.ofNullable(request.getHeader("X-Request-ID"))
                .filter(s -> !s.isBlank())
                .orElse(UUID.randomUUID().toString().substring(0, 8));

            MDC.put("requestId", requestId);
            response.setHeader("X-Request-ID", requestId);

            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### Add User Context

```java
// After authentication
MDC.put("userId", authentication.getName());

// All subsequent logs include userId automatically
log.info("User action performed");  // {"userId":"john123","message":"User action performed"}
```

### Context Propagation with Virtual Threads (Java 25+ / Spring Boot 3.2+)

MDC `ThreadLocal` context doesn't automatically propagate to new threads (including virtual threads). Modern Spring Boot uses **Micrometer Context Propagation**.

```java
import io.micrometer.context.ContextSnapshotFactory;
// ...

// ✅ Modern pattern: Wrap the task with context before submitting to an executor (or using @Async)
Runnable task = () -> {
    log.info("Async task running"); // Has requestId, userId
};

// Capture context from current thread
Runnable wrappedTask = ContextSnapshotFactory.builder().build().captureAll().wrap(task);

// Run on virtual threads
Executors.newVirtualThreadPerTaskExecutor().submit(wrappedTask);

// Alternatively, configure Spring's TaskExecutor to automatically propagate:
// spring.threads.virtual.enabled=true
// and use @Async
```

**Legacy pattern (Manual MDC copy):**

```java
// ⚠️ Manual copying is error-prone. Avoid in modern apps.
Map<String, String> context = MDC.getCopyOfContextMap();
CompletableFuture.runAsync(() -> {
    try {
        if (context != null) MDC.setContextMap(context);
        log.info("Async task running");
    } finally {
        MDC.clear();
    }
});
```

---

## What to Log

### Business Events (INFO)

```java
// Include key identifiers and state
log.atInfo()
   .setMessage("Order created")
   .addKeyValue("orderId", id)
   .addKeyValue("userId", userId)
   .addKeyValue("total", total)
   .addKeyValue("itemCount", items.size())
   .addKeyValue("step", "order_created")
   .log();

log.atInfo()
   .setMessage("Payment processed")
   .addKeyValue("orderId", id)
   .addKeyValue("amount", amount)
   .addKeyValue("method", "card")
   .addKeyValue("step", "payment_completed")
   .log();
```

### External Calls (with timing)

```java
long start = System.currentTimeMillis();
try {
    Result result = externalService.call(params);
    log.atInfo()
       .setMessage("External call succeeded")
       .addKeyValue("service", "PaymentGateway")
       .addKeyValue("operation", "charge")
       .addKeyValue("duration_ms", System.currentTimeMillis() - start)
       .log();
    return result;
} catch (Exception e) {
    log.atError()
       .setMessage("External call failed")
       .addKeyValue("service", "PaymentGateway")
       .addKeyValue("operation", "charge")
       .addKeyValue("duration_ms", System.currentTimeMillis() - start)
       .setCause(e)
       .log();
    throw e;
}
```

### Flow Steps (for AI tracing)

```java
public Order processOrder(CreateOrderRequest request) {
    log.atInfo().setMessage("Processing started").addKeyValue("step", "start").addKeyValue("requestData", request.summary()).log();

    Order order = createOrder(request);
    log.atInfo().setMessage("Order created").addKeyValue("step", "order_created").addKeyValue("orderId", order.getId()).log();

    validateInventory(order);
    log.atInfo().setMessage("Inventory validated").addKeyValue("step", "inventory_ok").addKeyValue("orderId", order.getId()).log();

    processPayment(order);
    log.atInfo().setMessage("Payment processed").addKeyValue("step", "payment_done").addKeyValue("orderId", order.getId()).log();

    log.atInfo().setMessage("Processing completed").addKeyValue("step", "complete").addKeyValue("orderId", order.getId()).log();
    return order;
}
```

---

## What NOT to Log

```java
// ❌ NEVER log sensitive data
log.atInfo().setMessage("Login").addKeyValue("password", password).log();           // Passwords
log.atInfo().setMessage("Payment").addKeyValue("cardNumber", card).log();           // Full card numbers
log.atInfo().setMessage("Request").addKeyValue("token", jwtToken).log();            // Tokens
log.atInfo().setMessage("User").addKeyValue("ssn", socialSecurity).log();           // PII

// ✅ Safe alternatives
log.atInfo().setMessage("Login attempted").addKeyValue("userId", userId).log();
log.atInfo().setMessage("Payment").addKeyValue("cardLast4", last4).log();
log.atInfo().setMessage("Token validated").addKeyValue("subject", sub).addKeyValue("exp", expiry).log();
```

---

## Exception Logging

### Log Once at Boundary

```java
// ❌ BAD: Logs same exception multiple times
void methodA() {
    try { methodB(); }
    catch (Exception e) { log.error("Error", e); throw e; }  // Log #1
}
void methodB() {
    try { methodC(); }
    catch (Exception e) { log.error("Error", e); throw e; }  // Log #2
}

// ✅ GOOD: Log at service boundary only
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handle(Exception e, HttpServletRequest request) {
        log.atError()
           .setMessage("Request failed")
           .addKeyValue("path", request.getRequestURI())
           .addKeyValue("method", request.getMethod())
           .addKeyValue("errorType", e.getClass().getSimpleName())
           .setCause(e)
           .log();  // Full stack trace
        return ResponseEntity.status(500).body(errorResponse);
    }
}
```

### Include Context

```java
// ❌ Useless
log.error("Error occurred", e);

// ✅ Useful for debugging
log.atError()
   .setMessage("Order processing failed")
   .addKeyValue("orderId", orderId)
   .addKeyValue("step", "payment")
   .addKeyValue("userId", userId)
   .addKeyValue("attemptNumber", attempt)
   .setCause(e)
   .log();
```

---

## Quick Reference

```java
// === Setup ===
private static final Logger log = LoggerFactory.getLogger(MyClass.class);

// === Logging with structured fields (SLF4J 2 API) ===
log.atInfo()
   .setMessage("Event")
   .addKeyValue("key1", value1)
   .addKeyValue("key2", value2)
   .log();

log.atError()
   .setMessage("Failed")
   .addKeyValue("context", ctx)
   .setCause(exception)
   .log();

// === MDC ===
MDC.put("requestId", requestId);
MDC.put("userId", userId);
// ... all logs now include these
MDC.clear();  // cleanup

// === Levels ===
log.error()  // Failures
log.warn()   // Handled issues
log.info()   // Business events
log.debug()  // Technical details
```

---

## Analyzing Logs (AI/Human)

```bash
# Pretty print JSON logs
tail -f app.log | jq .

# Filter errors
cat app.log | jq 'select(.level == "ERROR")'

# Follow request flow
cat app.log | jq 'select(.requestId == "abc123")'

# Find slow operations (>1s)
cat app.log | jq 'select(.duration_ms > 1000)'

# Get timeline of steps
cat app.log | jq 'select(.requestId == "abc123") | {time: .timestamp, step: .step, message: .message}'
```

---

## Related Skills

- `spring-boot-patterns` - Spring Boot configuration
- `jpa-patterns` - Database logging (SQL queries)
- Future: `observability-patterns` - Metrics, tracing, full observability
