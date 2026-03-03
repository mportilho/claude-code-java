---
name: concurrency-review
description: Review Java concurrency code for thread safety, race conditions, deadlocks, and modern patterns. Uses JDK 21 LTS as baseline with JDK 24/25 notes (Virtual Threads, CompletableFuture, @Async). Request via "check thread safety", "concurrency review".
---

# Concurrency Review Skill

Review Java concurrent code for correctness, safety, and modern best practices.

## Why This Matters

Concurrency bugs are:

- **Hard to reproduce** - timing-dependent
- **Hard to test** - may only appear under load
- **Hard to debug** - non-deterministic behavior

This skill helps catch issues **before** they reach production.

## When to Use

- Reviewing code with `synchronized`, `volatile`, `Lock`
- Checking `@Async`, `CompletableFuture`, `ExecutorService`
- Validating thread safety of shared state
- Reviewing Virtual Threads / Structured Concurrency code
- Any code accessed by multiple threads

---

## Modern Java Baseline: JDK 21 LTS (with notes for JDK 24/25)

### Virtual Threads

### When to Use Virtual Threads

```java
// ✅ Perfect for I/O-bound tasks (HTTP, DB, file I/O)
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request request : requests) {
        executor.submit(() -> callExternalApi(request));
    }
}

// ❌ Not beneficial for CPU-bound tasks
// Use platform threads / ForkJoinPool instead
```

**Rule of thumb**: Prefer virtual threads for blocking/I-O-heavy workloads;
validate with load tests when throughput/latency is critical.

Do not pool virtual threads to limit concurrency. For backpressure/limits,
keep per-task virtual threads and use explicit limiters (like `Semaphore`).

### JDK 24+: `synchronized` Pinning Mostly Fixed

In JDK 21-23, virtual threads could be "pinned" when blocking
inside `synchronized`. JDK 24 improves this significantly via JEP 491.

```java
// In Java 21-23: ⚠️ Could cause pinning
synchronized (lock) {
    blockingIoCall();  // Virtual thread pinned to carrier
}

// In JDK 24+: ✅ Most pinning cases removed
// Still keep critical sections small and avoid long blocking work while locked
```

Remaining pinning situations can still happen (e.g. native/FFM callback paths),
so monitor with JFR in production-critical flows.

### ScopedValue over ThreadLocal

```java
// ❌ ThreadLocal can hide context flow and is mutable
private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

// ✅ ScopedValue provides bounded, immutable context propagation
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

ScopedValue.where(CURRENT_USER, user).run(() -> {
    // CURRENT_USER.get() available here and in child virtual threads
    processRequest();
});
```

`ScopedValue` is preview in JDK 21-24 and finalized in JDK 25 (JEP 506).

### Structured Concurrency (Preview across JDK 21-25)

JDK 21-24 style (constructor-based):

```java
// ✅ Structured concurrency - subtasks tied to lexical scope
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    StructuredTaskScope.Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    StructuredTaskScope.Subtask<Orders> ordersTask =
        scope.fork(() -> fetchOrders(id));

    scope.join();            // Wait for all
    scope.throwIfFailed();   // Propagate exceptions

    return new Profile(userTask.get(), ordersTask.get());
}
```

JDK 25 style (factory-based, JEP 505):

```java
try (var scope = StructuredTaskScope.open()) {
    StructuredTaskScope.Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    StructuredTaskScope.Subtask<Orders> ordersTask = scope.fork(() -> fetchOrders(id));

    scope.join();  // Default policy: fail-fast on subtask failure
    return new Profile(userTask.get(), ordersTask.get());
}
```

Track JDK 26 if your runtime roadmap includes it (`StructuredTaskScope`
remains preview and may evolve again).

---

## Spring @Async Pitfalls

### 1. Forgetting @EnableAsync

```java
// ❌ @Async silently ignored
@Service
public class EmailService {
    @Async
    public void sendEmail(String to) { }
}

// ✅ Enable async processing
@Configuration
@EnableAsync
public class AsyncConfig { }
```

### 2. Calling Async from Same Class

```java
@Service
public class OrderService {

    // ❌ Bypasses proxy - runs synchronously!
    public void processOrder(Order order) {
        sendConfirmation(order);  // Direct call, not async
    }

    @Async
    public void sendConfirmation(Order order) { }
}

// ✅ Inject self or use separate service
@Service
public class OrderService {
    @Autowired
    private EmailService emailService;  // Separate bean

    public void processOrder(Order order) {
        emailService.sendConfirmation(order);  // Proxy call, async works
    }
}
```

### 3. @Async Proxy Boundaries (Visibility and Finality)

```java
// ❌ Private/final methods cannot be advised by Spring proxies
@Async
private void processInBackground() { }

@Async
public final void processInBackground() { }

// ✅ Prefer non-final methods invoked through a Spring bean proxy
@Async
public void processInBackground() { }
```

`@Async` interception is proxy-based: local calls (same-instance
`this.method()`) bypass async behavior. Call via injected beans.

### 4. Default Executor Behavior Depends on Stack

```java
// ⚠️ In plain Spring Framework, if no Executor bean exists,
// fallback is often SimpleAsyncTaskExecutor (new thread per task).
// In Spring Boot earlier than 3.2, default auto-configuration was ThreadPoolTaskExecutor.

// ✅ In Spring Boot 4 (and 3.2+), the modern approach is to enable virtual threads.
// This auto-configures a task executor optimized for I/O bounds automatically.
// In your application.properties / .yml:
// spring.threads.virtual.enabled=true

// ✅ If explicitly needing custom thread pool policy for CPU bounds (non-virtual):
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("cpuBoundTaskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("cpu-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### 5. Context Propagation (Security, Logging, Tracing)

```java
// ❌ SecurityContextHolder / MDC are ThreadLocal-bound
@Async
public void auditAction() {
    // SecurityContextHolder.getContext() is NULL here!
    String user = SecurityContextHolder.getContext().getAuthentication().getName();
}

// ✅ Use Micrometer Context Propagation with Spring Boot 4 / Framework 7+
// Spring Boot auto-configures context propagation for @Async and Observability 
// when spring.threads.virtual.enabled=true or context propagation is on the classpath.
@Async
public void auditAction() {
    // Context is automatically propagated by Spring's modern abstractions!
    String user = SecurityContextHolder.getContext().getAuthentication().getName();
}

// ✅ Legacy workaround (Spring Security 5.x / older versions):
// Return new DelegatingSecurityContextAsyncTaskExecutor(executor);
```

---

## CompletableFuture Patterns

### Error Handling

```java
// Assuming a standard SLF4J Logger setup:
// private static final Logger logger = LoggerFactory.getLogger(MyClass.class);

// ❌ Exception silently swallowed
CompletableFuture.supplyAsync(() -> riskyOperation());
// If riskyOperation throws, nobody knows

// ✅ Always handle exceptions
CompletableFuture.supplyAsync(() -> riskyOperation())
    .exceptionally(ex -> {
        logger.error("Operation failed", ex);
        return fallbackValue;
    });

// ✅ Or use handle() for both success and failure
CompletableFuture.supplyAsync(() -> riskyOperation())
    .handle((result, ex) -> {
        if (ex != null) {
            logger.error("Failed", ex);
            return fallbackValue;
        }
        return result;
    });
```

### Timeout Handling (Java 9+)

```java
// ✅ Fail after timeout
CompletableFuture.supplyAsync(() -> slowOperation())
    .orTimeout(5, TimeUnit.SECONDS);  // Throws TimeoutException

// ✅ Return default after timeout
CompletableFuture.supplyAsync(() -> slowOperation())
    .completeOnTimeout(defaultValue, 5, TimeUnit.SECONDS);
```

### Combining Futures

```java
// ✅ Wait for all
CompletableFuture.allOf(future1, future2, future3)
    .thenRun(() -> logger.info("All completed"));

// ✅ Wait for first
CompletableFuture.anyOf(future1, future2, future3)
    .thenAccept(result -> logger.info("First result: {}", result));

// ✅ Combine results
future1.thenCombine(future2, (r1, r2) -> merge(r1, r2));
```

### Use Appropriate Executor

```java
// ⚠️ Implicit commonPool: no workload isolation or explicit capacity policy
CompletableFuture.supplyAsync(() -> maybeBlockingCall());

// ✅ Custom executor for blocking/I/O operations
ExecutorService ioExecutor = Executors.newFixedThreadPool(20);
CompletableFuture.supplyAsync(() -> blockingIoCall(), ioExecutor);

// ✅ In Java 21+, virtual threads for I/O
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();
CompletableFuture.supplyAsync(() -> blockingIoCall(), virtualExecutor);

// ✅ Dedicated CPU executor for CPU-bound work
ExecutorService cpuExecutor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);
CompletableFuture.supplyAsync(() -> cpuIntensiveWork(), cpuExecutor);
```

---

## Classic Concurrency Issues

### Race Conditions: Check-Then-Act

```java
// ❌ Race condition
if (!map.containsKey(key)) {
    map.put(key, computeValue());  // Another thread may have added it
}

// ✅ Atomic operation
map.computeIfAbsent(key, k -> computeValue());

// ❌ Race condition with counter
if (count < MAX) {
    count++;  // Read-check-write is not atomic
}

// ✅ Atomic counter
AtomicInteger count = new AtomicInteger();
count.updateAndGet(c -> c < MAX ? c + 1 : c);
```

### Visibility: Missing volatile

```java
// ❌ Other threads may never see the update
private boolean running = true;

public void stop() {
    running = false;  // May not be visible to other threads
}

public void run() {
    while (running) { }  // May loop forever
}

// ✅ Volatile ensures visibility
private volatile boolean running = true;
```

### Non-Atomic long/double

```java
// ❌ 64-bit read/write is non-atomic on 32-bit JVMs
private long counter;

public void increment() {
    counter++;  // Not atomic!
}

// ✅ Use AtomicLong or synchronization
private AtomicLong counter = new AtomicLong();

// ✅ Or volatile (for single-writer scenarios)
private volatile long counter;
```

### Double-Checked Locking

```java
// ❌ Broken without volatile
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();  // May be seen partially constructed
            }
        }
    }
    return instance;
}

// ✅ Correct with volatile
private static volatile Singleton instance;

// ✅ Or use holder class idiom
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}

public static Singleton getInstance() {
    return Holder.INSTANCE;
}
```

### Deadlocks: Lock Ordering

```java
// ❌ Potential deadlock
// Thread 1: lock(A) -> lock(B)
// Thread 2: lock(B) -> lock(A)

public void transfer(Account from, Account to, int amount) {
    synchronized (from) {
        synchronized (to) {
            // Transfer logic
        }
    }
}

// ✅ Consistent lock ordering
public void transfer(Account from, Account to, int amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized (first) {
        synchronized (second) {
            // Transfer logic
        }
    }
}
```

---

## Thread-Safe Collections

### Choose the Right Collection

| Use Case | Wrong | Right |
| :--- | :--- | :--- |
| Concurrent reads/writes | `HashMap` | `ConcurrentHashMap` |
| Frequent iteration | `ConcurrentHashMap` | `CopyOnWriteArrayList` |
| Producer-consumer | `ArrayList` | `BlockingQueue` |
| Sorted concurrent | `TreeMap` | `ConcurrentSkipListMap` |

### ConcurrentHashMap Pitfalls

```java
// ❌ Non-atomic compound operation
if (!map.containsKey(key)) {
    map.put(key, value);
}

// ✅ Atomic
map.putIfAbsent(key, value);
map.computeIfAbsent(key, k -> createValue());

// ❌ Mapping functions must not update this map
// ConcurrentHashMap may throw IllegalStateException on detectable recursive updates
map.compute(key1, (k, v) -> {
    map.put(key2, otherValue);
    return v;
});
```

---

## Concurrency Review Checklist

### 🔴 High Severity (Likely Bugs)

- [ ] No check-then-act on shared state without synchronization
- [ ] No blocking/external calls while holding `synchronized`/locks
- [ ] `volatile` present for double-checked locking and stop flags
- [ ] `ConcurrentHashMap` mapping functions do not mutate the same map
- [ ] `@Async` methods are interceptable by proxy and called through beans

### 🟡 Medium Severity (Potential Issues)

- [ ] Thread pools are sized, named, and have a rejection policy
- [ ] CompletableFuture chains handle exceptions and set timeouts
- [ ] Context Propagated (Micrometer) enabled for Security/Tracing in async
- [ ] `ExecutorService` shutdown and `Lock.unlock()` in `finally`
- [ ] Thread-safe collections used for shared data

### 🟢 Modern Patterns (JDK 21 baseline + notes for 24/25)

- [ ] Virtual threads used for I/O tasks (validated via Boot 4 auto config)
- [ ] ScopedValue considered over ThreadLocal for request context
- [ ] Structured concurrency used for related subtasks/lifecycle control

### 📝 Documentation

- [ ] Thread safety documented on shared classes
- [ ] Locking order and synchronization boundaries documented

---

## Analysis Commands

```bash
# Find synchronized blocks
rg -n --glob '*.java' "synchronized" .

# Find @Async methods
rg -n --glob '*.java' "@Async" .

# Find volatile fields
rg -n --glob '*.java' "volatile" .

# Find thread pool creation
rg -n --glob '*.java' "Executors\\.|ThreadPoolExecutor|ExecutorService" .

# Find CompletableFuture without error handling
rg -n --glob '*.java' "CompletableFuture\\." . | rg -v "exceptionally|handle|whenComplete|orTimeout|completeOnTimeout"

# Find ThreadLocal (consider ScopedValue in Java 21+)
rg -n --glob '*.java' "ThreadLocal" .
```
