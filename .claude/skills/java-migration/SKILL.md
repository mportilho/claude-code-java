---
name: java-migration
description: Guide for upgrading Java projects between major versions (8→11→17→21→25). Use when user says "upgrade Java", "migrate to Java 25", "update Java version", or when modernizing legacy projects.
---

# Java Migration Skill

Step-by-step guide for upgrading Java projects between major versions.

## When to Use
- User says "upgrade to Java 25" / "migrate from Java 8" / "update Java version"
- Modernizing legacy projects
- Spring Boot 2.x → 3.x → 4.x migration
- Preparing for LTS version adoption

## Migration Paths

```
Java 8 (LTS) → Java 11 (LTS) → Java 17 (LTS) → Java 21 (LTS) → Java 25 (LTS)
     │              │               │              │               │
     └──────────────┴───────────────┴──────────────┴───────────────┘
                         Always migrate LTS → LTS
```

---

## Quick Reference: What Breaks

| From → To | Major Breaking Changes |
|-----------|------------------------|
| 8 → 11 | Removed `javax.xml.bind`, module system, internal APIs |
| 11 → 17 | Sealed classes (preview→final), strong encapsulation |
| 17 → 21 | Pattern matching changes, `finalize()` deprecated for removal |
| 21 → 25 | Security Manager permanently disabled, 32-bit x86 removed, new warnings for Unsafe memory-access methods and dynamic agents |

---

## Migration Workflow

### Step 1: Assess Current State

```bash
# Check current Java version
java -version

# Check compiler target in Maven
grep -r "maven.compiler" pom.xml

# Find usage of removed APIs
grep -r "sun\." --include="*.java" src/
grep -r "javax\.xml\.bind" --include="*.java" src/
```

### Step 2: Update Build Configuration

**Maven:**
```xml
<properties>
    <java.version>21</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
</properties>

<!-- Or with compiler plugin -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <release>21</release>
    </configuration>
</plugin>
```

**Gradle:**
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

### Step 3: Fix Compilation Errors

Run compile and fix errors iteratively:
```bash
mvn clean compile 2>&1 | head -50
```

### Step 4: Run Tests

```bash
mvn test
```

### Step 5: Check Runtime Warnings

```bash
# Scan for JDK internals in your artifacts
jdeps --jdk-internals --recursive target/*.jar

# Scan APIs deprecated for removal
jdeprscan --release 25 --for-removal target/*.jar
```

---

## Java 8 → 11 Migration

### Removed APIs

| Removed | Replacement |
|---------|-------------|
| `javax.xml.bind` (JAXB) | Add dependency: `jakarta.xml.bind-api` + `jaxb-runtime` |
| `javax.activation` | Add dependency: `jakarta.activation-api` |
| `javax.annotation` | Add dependency: `jakarta.annotation-api` |
| `java.corba` | No replacement (rarely used) |
| `java.transaction` | Add dependency: `jakarta.transaction-api` |
| `sun.misc.Base64*` | Use `java.util.Base64` |
| `sun.misc.Unsafe` (partially) | Use `VarHandle` where possible |

### Add Missing Dependencies (Maven)

```xml
<!-- JAXB (if needed) -->
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>4.0.1</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.4</version>
    <scope>runtime</scope>
</dependency>

<!-- Annotation API -->
<dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>2.1.1</version>
</dependency>
```

### Module System Issues

If using reflection on JDK internals, add JVM flags:
```bash
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
```

**Maven Surefire:**
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

### New Features to Adopt

```java
// var (local variable type inference)
var list = new ArrayList<String>();  // instead of ArrayList<String> list = ...

// String methods
"  hello  ".isBlank();      // true for whitespace-only
"  hello  ".strip();        // better trim() (Unicode-aware)
"line1\nline2".lines();     // Stream<String>
"ha".repeat(3);             // "hahaha"

// Collection factory methods (Java 9+)
List.of("a", "b", "c");     // immutable list
Set.of(1, 2, 3);            // immutable set
Map.of("k1", "v1");         // immutable map

// Optional improvements
optional.ifPresentOrElse(
    value -> process(value),
    () -> handleEmpty()
);

// HTTP Client (replaces HttpURLConnection)
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com"))
    .build();
HttpResponse<String> response = client.send(request,
    HttpResponse.BodyHandlers.ofString());
```

---

## Java 11 → 17 Migration

### Breaking Changes

| Change | Impact |
|--------|--------|
| Strong encapsulation | `--illegal-access` no longer works, must use explicit `--add-opens` |
| Sealed classes (final) | If you used preview features |
| Pattern matching instanceof | Preview → final syntax change |

### New Features to Adopt

```java
// Records (immutable data classes)
public record User(String name, String email) {}
// Auto-generates: constructor, getters, equals, hashCode, toString

// Sealed classes
public sealed class Shape permits Circle, Rectangle {}
public final class Circle extends Shape {}
public final class Rectangle extends Shape {}

// Pattern matching for instanceof
if (obj instanceof String s) {
    System.out.println(s.length());  // s already cast
}

// Switch expressions
String result = switch (day) {
    case MONDAY, FRIDAY -> "Work";
    case SATURDAY, SUNDAY -> "Rest";
    default -> "Midweek";
};

// Text blocks
String json = """
    {
        "name": "John",
        "age": 30
    }
    """;

// Helpful NullPointerException messages
// a.b.c.d() → tells exactly which part was null
```

---

## Java 17 → 21 Migration

### Breaking Changes

| Change | Impact |
|--------|--------|
| Pattern matching switch (final) | Minor syntax differences from preview |
| `finalize()` deprecated for removal | Replace with `Cleaner` or try-with-resources |
| UTF-8 by default | May affect file reading if assumed platform encoding |

### New Features to Adopt

```java
// Virtual Threads (Project Loom) - MAJOR
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> handleRequest());
}
// Or simply:
Thread.startVirtualThread(() -> doWork());

// Pattern matching in switch
String formatted = switch (obj) {
    case Integer i -> "int: " + i;
    case String s -> "string: " + s;
    case null -> "null value";
    default -> "unknown";
};

// Record patterns
record Point(int x, int y) {}
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);
}

// Sequenced Collections
List<String> list = new ArrayList<>();
list.addFirst("first");    // new method
list.addLast("last");      // new method
list.reversed();           // reversed view

// Scoped Values (preview) - replace ThreadLocal
ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // CURRENT_USER.get() available here
});
```

---

## Java 21 → 25 Migration

### Breaking Changes

| Change | Impact |
|--------|--------|
| Security Manager permanently disabled | Enabling it at startup/runtime is no longer supported (JDK 24+) |
| `sun.misc.Unsafe` memory-access methods terminally deprecated | Runtime warnings in JDK 24+; migrate to `VarHandle`/FFM API |
| 32-bit x86 port removed | OpenJDK no longer supports 32-bit x86 in JDK 25 |
| `ScopedValue.orElse(null)` disallowed | Must provide non-null default |
| Dynamic agent loading warns by default | Use startup agents (`-javaagent`) or explicit opt-in when required |

### Check for Unsafe Usage

```bash
# Find sun.misc.Unsafe usage
grep -rn "sun\.misc\.Unsafe" --include="*.java" src/

# Find terminally deprecated APIs in build artifacts
jdeprscan --release 25 --for-removal target/*.jar

# Find Security Manager usage
grep -rn "SecurityManager\|System\.getSecurityManager" --include="*.java" src/
```

### New Features to Adopt

```java
// Scoped Values (FINAL in Java 25) - replaces ThreadLocal
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

public void handleRequest(User user) {
    ScopedValue.where(CURRENT_USER, user).run(() -> {
        processRequest();  // CURRENT_USER.get() available here and in child threads
    });
}

// Structured Concurrency (Preview, redesigned API in 25)
try (var scope = StructuredTaskScope.open(StructuredTaskScope.Joiner.awaitAllSuccessfulOrThrow())) {
    StructuredTaskScope.Subtask<UserAccount> userTask = scope.fork(() -> fetchUser(id));
    StructuredTaskScope.Subtask<OrderSummary> ordersTask = scope.fork(() -> fetchOrders(id));

    scope.join(); // Throws on failure with this joiner
    return new CustomerProfile(userTask.get(), ordersTask.get());
}

// Stable Values (Preview) - lazy initialization made easy
private static final StableValue<ExpensiveService> SERVICE = StableValue.of();

public void useService() {
    SERVICE.orElseSet(ExpensiveService::new).doWork();  // Initialized on first access
}

// Compact Object Headers - product feature in Java 25 (opt-in)
// Enable with: -XX:+UseCompactObjectHeaders

// Primitive Patterns in instanceof (Preview)
if (obj instanceof int i) {
    System.out.println("int value: " + i);
}

// Module Import Declarations (final in Java 25)
import module java.sql;  // Import all public types from module
```

### Performance-Related Updates

Java 25 includes performance-related updates, but not all are enabled by default:
- **Compact Object Headers** is now a product feature and can be enabled with `-XX:+UseCompactObjectHeaders`.
- **Unsafe memory-access warnings** (JDK 24+) help identify libraries that still depend on unsupported APIs.
- Evaluate VM flags and runtime options with workload-specific benchmarks before production rollout.

### Migration with OpenRewrite

```bash
# Automated Java 25 migration
# Prefer pinning recipe versions in CI; use LATEST for exploratory runs
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.migrate.UpgradeToJava25
```

---

## Spring Boot Migration

### Spring Boot 2.x → 3.x

**Requirements:**
- Java 17+ (mandatory)
- Jakarta EE 9+ (javax.* → jakarta.*)

**Package Renames:**
```java
// Before (Spring Boot 2.x)
import javax.persistence.*;
import javax.validation.*;
import javax.servlet.*;

// After (Spring Boot 3.x)
import jakarta.persistence.*;
import jakarta.validation.*;
import jakarta.servlet.*;
```

**Find & Replace:**
```bash
# Find all javax imports that need migration
grep -r "import javax\." --include="*.java" src/ | grep -v "javax.crypto" | grep -v "javax.net"
```

**Automated migration:**
```bash
# Use OpenRewrite
# Prefer pinning recipe versions in CI; use LATEST for exploratory runs
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0
```

### Dependency Updates (Spring Boot 3.x)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.2</version>
</parent>

<!-- Hibernate 6 (auto-included) -->
<!-- Spring Security 6 (auto-included) -->
```

### Hibernate 5 → 6 Changes

```java
// ID generation strategy changed
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)  // preferred
private Long id;

// Query changes
// Before: createQuery returns raw type
// After: createQuery requires type parameter

// Before
Query query = session.createQuery("from User");

// After
TypedQuery<User> query = session.createQuery("from User", User.class);
```

---

## Spring Boot 3.x → 4.x

**Requirements:**
- Java 21+ (mandatory)
- Jakarta EE 11+ (jakarta.* updates)

**Key Changes:**
- Migration from Spring Framework 6 to 7
- Removal of deprecated features from 3.x
- Upgraded dependencies (Hibernate 7.x, Tomcat 11)

**Automated migration:**
```bash
# Use OpenRewrite
# Prefer pinning recipe versions in CI; use LATEST for exploratory runs
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_4_0
```
```

---

## Common Migration Issues

### Issue: Reflection Access Denied

**Symptom:**
```
java.lang.reflect.InaccessibleObjectException: Unable to make field accessible
```

**Fix:**
```bash
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
```

### Issue: JAXB ClassNotFoundException

**Symptom:**
```
java.lang.ClassNotFoundException: javax.xml.bind.JAXBContext
```

**Fix:** Add JAXB dependencies (see Java 8→11 section)

### Issue: Test Failures with Mockito

**Fix:** Update Mockito:
```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
```

---

## Migration Checklist

### Pre-Migration
- [ ] Document current Java version
- [ ] List all dependencies and their versions
- [ ] Identify usage of internal APIs (`sun.*`, `com.sun.*`)
- [ ] Check framework compatibility (Spring, Hibernate, etc.)
- [ ] Backup / create branch

### During Migration
- [ ] Update build tool configuration
- [ ] Add missing Jakarta dependencies
- [ ] Fix `javax.*` → `jakarta.*` imports (if Spring Boot 3)
- [ ] Add `--add-opens` flags if needed
- [ ] Update Mockito, other tools
- [ ] Fix compilation errors
- [ ] Run tests

### Post-Migration
- [ ] Remove unnecessary `--add-opens` flags
- [ ] Adopt new language features (records, var, etc.)
- [ ] Update CI/CD pipeline
- [ ] Document changes made

---

## Quick Commands

```bash
# Check Java version
java -version

# Find internal API usage
grep -rn "sun\.\|com\.sun\." --include="*.java" src/

# Find javax imports (for Jakarta migration)
grep -rn "import javax\." --include="*.java" src/

# Compile and show first errors
mvn clean compile 2>&1 | head -100

# Find dependencies on JDK internals
jdeps --jdk-internals --recursive target/*.jar

# Find APIs deprecated for removal
jdeprscan --release 25 --for-removal target/*.jar

# OpenRewrite Spring Boot 3 migration
mvn org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:LATEST \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0
```

---

## Version Compatibility Matrix

| Framework | Java 8 | Java 11 | Java 17 | Java 21 | Java 25 |
|-----------|--------|---------|---------|---------|---------|
| Spring Boot 2.7.x | ✅ | ✅ | ✅ | ⚠️ | ❌ |
| Spring Boot 3.2.x | ❌ | ❌ | ✅ | ✅ | ❌ |
| Spring Boot 4.x | ❌ | ❌ | ❌ | ✅ | ✅ |
| Hibernate 5.6 | ✅ | ✅ | ✅ | ⚠️ | ❌ |
| Hibernate 6.4+ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Hibernate 7.x | ❌ | ❌ | ❌ | ✅ | ✅ |
| JUnit 5.10+ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Mockito 5.x | ❌ | ✅ | ✅ | ✅ | ✅ |

**Oracle Java SE Roadmap (commercial support dates):**
- Java 21 (LTS): Premier Support until September 2028; Extended Support until September 2031
- Java 25 (LTS): Premier Support until September 2030; Extended Support until September 2033

