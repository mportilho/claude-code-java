# Concurrency Review Skill

> Review Java concurrent code for thread safety, race conditions, and modern patterns (JDK 21 LTS baseline with notes for JDK 24/25)

## What It Does

Reviews multi-threaded Java code for:
- Race conditions and visibility issues
- Deadlock potential
- Modern patterns (Virtual Threads, Structured Concurrency)
- Spring @Async pitfalls
- CompletableFuture error handling
- Thread pool configuration

## Why It Matters

Concurrency bugs are hard to reproduce, hard to test, and hard to debug. Catching them in code review is far better than finding them in production.

## When to Use

- "Review this for thread safety"
- "Check concurrency issues"
- "Is this async code correct?"
- Reviewing code with `synchronized`, `volatile`, `@Async`
- Checking `CompletableFuture` or `ExecutorService` usage

## Key Topics Covered

### Modern Java (JDK 21 baseline + notes for 24/25)
| Topic | What to Check |
|-------|---------------|
| Virtual Threads | Use for I/O-bound; avoid pooling Virtual Threads for throttling (use limiters) |
| Structured Concurrency | Proper scope management (preview; API differs by JDK) |
| ScopedValue | Prefer over ThreadLocal for bounded context |

### Spring @Async
| Pitfall | Issue |
|---------|-------|
| Same-class call | Bypasses proxy, runs sync |
| Proxy boundaries | Private/final or local calls bypass async interception |
| Default executor | Depends on stack (Spring Framework fallback vs Spring Boot auto-config) |
| SecurityContext | ThreadLocal doesn't propagate |

### Classic Issues
| Issue | Example |
|-------|---------|
| Race condition | Check-then-act without sync |
| Visibility | Missing volatile |
| Deadlock | Inconsistent lock ordering |

## Example Usage

```
You: Review this service for thread safety

Claude: [Checks shared mutable state]
        [Validates synchronization]
        [Reviews @Async configuration]
        [Checks CompletableFuture error handling]
        [Suggests modern alternatives if applicable]
```

## Severity Levels

| Level | Meaning |
|-------|---------|
| 🔴 High | Likely bug - race condition, deadlock risk |
| 🟡 Medium | Potential issue - measure/verify |
| 🟢 Modern | Opportunity for JDK 21+ modern concurrency patterns |

## Related Skills

- `performance-smell-detection` - Performance issues (not thread safety)
- `java-code-review` - General code review (includes basic concurrency)
- `spring-boot-patterns` - Spring patterns (includes @Async basics)

## References

- [OpenJDK JEP 444 - Virtual Threads](https://openjdk.org/jeps/444)
- [Java Concurrency Code Review Checklist](https://github.com/code-review-checklists/java-concurrency)
- [Baeldung - Common Concurrency Pitfalls](https://www.baeldung.com/java-common-concurrency-pitfalls)
- [Oracle - Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [OpenJDK JEP 491 - Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [OpenJDK JEP 505 - Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505)
- [OpenJDK JEP 506 - Scoped Values](https://openjdk.org/jeps/506)
- [Spring Framework `@EnableAsync` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableAsync.html)
- [Spring Boot Task Execution and Scheduling](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)
- Book: "Java Concurrency in Practice" by Brian Goetz
