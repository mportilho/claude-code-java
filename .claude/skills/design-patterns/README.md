# Design Patterns

**Load**: `view .claude/skills/design-patterns/SKILL.md`

---

## Description

Common design patterns with practical Java examples. Covers creational, behavioral, and structural patterns with modern Java syntax and Spring integration.

---

## Use Cases

- "Implement factory pattern for notifications"
- "Use builder for this complex object"
- "How to add features without modifying class?" (Decorator)
- "Multiple payment methods, swap at runtime" (Strategy)
- "Notify multiple services when order placed" (Observer)

---

## Examples

```
> view .claude/skills/design-patterns/SKILL.md
> "I need to create different report types (PDF, Excel, CSV)"
→ Suggests Factory pattern with implementation example
```

---

## Patterns Covered

| Category | Patterns |
|----------|----------|
| **Creational** | Builder, Factory (Simple/Static), Singleton |
| **Behavioral** | Strategy, Observer, Template Method |
| **Structural** | Decorator, Adapter |
| **Cross-cutting** | Declarative Resilience (Spring 7+) |

---

## Quick Selection Guide

| Problem | Pattern |
|---------|---------|
| Many constructor parameters | Builder |
| Create without specifying class | Factory |
| Swap algorithms at runtime | Strategy |
| Add behavior dynamically | Decorator |
| Notify multiple objects | Observer |
| Integrate legacy code | Adapter |
| Guard remote calls with retry/limits | Declarative Resilience |

---

## Related Skills

- `solid-principles` - Principles patterns implement
- `clean-code` - Code-level practices
- `spring-boot-patterns` - Spring implementations

---

## Resources

- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)
- [Design Patterns by Gang of Four](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/)
- [Java Design Patterns (java-design-patterns.com)](https://java-design-patterns.com/)
- [JDK 25 - Observer (Deprecated)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Observer.html)
- [JDK 25 - Observable (Deprecated)](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Observable.html)
- [Spring Framework - Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Spring Framework - Application Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html)
- [Spring Framework - Resilience Features](https://docs.spring.io/spring-framework/reference/core/resilience.html)
