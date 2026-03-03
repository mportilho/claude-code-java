---
name: maven-dependency-audit
description: Audit Maven dependencies for outdated versions, security vulnerabilities, and conflicts. Use when user says "check dependencies", "audit dependencies", "outdated deps", or before releases.
---

# Maven Dependency Audit Skill

Audit Maven dependencies for updates, vulnerabilities, and conflicts.

## When to Use
- User says "check dependencies" / "audit dependencies" / "outdated dependencies"
- Before a release
- Regular maintenance (monthly recommended)
- After security advisory
- During migration to Java 21/25 or Spring Boot 4

## Audit Workflow

1. **Check for updates** - Find outdated dependencies
2. **Analyze tree** - Find conflicts and duplicates
3. **Security scan & SBOM** - Check for vulnerabilities and generate bill of materials
4. **Report** - Summary with prioritized actions

---

## 1. Check for Outdated Dependencies

### Command
```bash
mvn versions:display-dependency-updates
```

### Output Analysis
```
[INFO] The following dependencies in Dependencies have newer versions:
[INFO]   org.slf4j:slf4j-api ......................... 2.0.9 -> 2.0.16
[INFO]   com.fasterxml.jackson.core:jackson-databind . 2.16.1 -> 2.21.1
[INFO]   org.junit.jupiter:junit-jupiter ............. 5.10.1 -> 5.12.0
```

### Categorize Updates

| Category | Criteria | Action |
|----------|----------|--------|
| **Security** | CVE fix in newer version | Update ASAP |
| **Major** | x.0.0 change | Review changelog, test thoroughly |
| **Minor** | x.y.0 change | Usually safe, test |
| **Patch** | x.y.z change | Safe, minimal testing |

> [!CAUTION]
> **Spring Boot 4 / Managed Dependencies:** Do **not** manually update versions of transitive dependencies managed by Spring Boot's dependency management (`spring-boot-dependencies` BOM). If a managed dependency is outdated, update the Spring Boot parent version instead to avoid unpredictable compatibility issues. Only manually bump versions if you explicitly need to override a managed version due to a zero-day CVE, and do so specifying the version in `<properties>`.

### Check Plugin Updates Too
```bash
mvn versions:display-plugin-updates
```

---

## 2. Analyze Dependency Tree

### Full Tree
```bash
mvn dependency:tree
```

### Filter for Specific Dependency
```bash
mvn dependency:tree -Dincludes=org.slf4j
```

### Find Conflicts
Look for:
```
[INFO] +- com.example:module-a:jar:1.0:compile
[INFO] |  \- org.slf4j:slf4j-api:jar:2.0.9:compile
[INFO] +- com.example:module-b:jar:1.0:compile
[INFO] |  \- org.slf4j:slf4j-api:jar:2.0.16:compile (omitted for conflict)
```

**Flags:**
- `(omitted for conflict)` - Version conflict resolved by Maven
- `(omitted for duplicate)` - Same version, no issue
- Multiple versions of same library - Potential runtime issues

### Analyze Unused Dependencies
```bash
mvn dependency:analyze
```

Output:
```
[WARNING] Used undeclared dependencies found:
[WARNING]    org.slf4j:slf4j-api:jar:2.0.16:compile
[WARNING] Unused declared dependencies found:
[WARNING]    commons-io:commons-io:jar:2.15.1:compile
```

---

## 3. Security Vulnerability Scan & SBOM

### Option A: OWASP Dependency-Check (Recommended for Vulnerabilities)

Add to `pom.xml`:
```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>12.2.0</version>
</plugin>
```

Run:
```bash
mvn dependency-check:check
```

Output: HTML report in `target/dependency-check-report.html`

### Option B: SBOM Generation (Modern Best Practice)

Always generate a Software Bill of Materials (SBOM) for mature projects, providing transparency for supply chain security.

Add to `pom.xml`:
```xml
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.9.1</version>
</plugin>
```

Run to generate SBOM:
```bash
mvn cyclonedx:makeAggregateBom
```
Output: SBOM in `target/bom.json` and `target/bom.xml`.

### Option C: Maven Dependency Plugin
```bash
mvn dependency:analyze-report
```

### Option D: GitHub Dependabot
If using GitHub, enable Dependabot alerts in repository settings.

### Severity Levels

| CVSS Score | Severity | Action |
|------------|----------|--------|
| 9.0 - 10.0 | Critical | Update immediately |
| 7.0 - 8.9 | High | Update within days |
| 4.0 - 6.9 | Medium | Update within weeks |
| 0.1 - 3.9 | Low | Update at convenience |

---

## 4. Generate Audit Report

### Output Format

```markdown
## Dependency Audit Report

**Project:** {project-name}
**Date:** {date}
**Total Dependencies:** {count}

### Security Issues

| Dependency | Current | CVE | Severity | Fixed In |
|------------|---------|-----|----------|----------|
| log4j-core | 2.14.0 | CVE-2021-44228 | Critical | 2.17.1 |

### Outdated Dependencies

#### Major Updates (Review Required)
| Dependency | Current | Latest | Notes |
|------------|---------|--------|-------|
| guava | 32.1.3-jre | 33.0.0-jre | Review changelog |

#### Minor/Patch Updates (Safe)
| Dependency | Current | Latest |
|------------|---------|--------|
| junit-jupiter | 5.10.1 | 5.12.0 |
| jackson-databind | 2.16.1 | 2.21.1 |
| slf4j-api | 2.0.9 | 2.0.16 |

### Conflicts Detected
- slf4j-api: 2.0.9 vs 2.0.16 (resolved to 2.0.16)

### Unused Dependencies
- commons-io:commons-io:2.15.1 (consider removing)

### Recommendations
1. **Immediate:** Update log4j-core to fix CVE-2021-44228
2. **This sprint:** Update minor/patch versions
3. **Security:** Upload generated `sbom.json` to tracking system
```

---

## Common Scenarios

### Scenario: Check Before Release
```bash
# Quick check
mvn versions:display-dependency-updates -q

# Full audit
mvn versions:display-dependency-updates && \
mvn dependency:analyze && \
mvn dependency-check:check
```

### Scenario: Find Why Dependency is Included
```bash
mvn dependency:tree -Dincludes=commons-logging
```

### Scenario: Force Specific Version (Resolve Conflict)
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Scenario: Exclude Transitive Dependency
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>some-library</artifactId>
    <version>1.0</version>
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

> [!IMPORTANT]
> **JDK 21/25 Native Plugins:** When targeting modern JDKs like JDK 21 or JDK 25, heavily audit Maven core plugins (e.g., `maven-compiler-plugin`, `maven-surefire-plugin`, `maven-failsafe-plugin`). Outdated plugins may fail to parse or compile modern Java class bytecode versions. Always ensure they are updated to recent releases.

---

## Token Optimization

- Use `-q` (quiet) flag for less verbose output
- Filter with `-Dincludes=groupId:artifactId` when looking for specific deps
- Run commands separately and summarize findings
- Don't paste entire dependency tree - summarize conflicts

## Quick Commands Reference

| Task | Command |
|------|---------|
| Outdated deps | `mvn versions:display-dependency-updates` |
| Outdated plugins | `mvn versions:display-plugin-updates` |
| Dependency tree | `mvn dependency:tree` |
| Find specific dep | `mvn dependency:tree -Dincludes=groupId` |
| Unused deps | `mvn dependency:analyze` |
| Security scan | `mvn dependency-check:check` |
| Update versions | `mvn versions:use-latest-releases` |
| Update snapshots | `mvn versions:use-latest-snapshots` |

## Update Strategies

### Conservative (Recommended for Production)
1. Update patch versions freely
2. Update minor versions with basic testing
3. Major versions require migration plan

### Aggressive (For Active Development)
```bash
# Update all to latest (use with caution!)
mvn versions:use-latest-releases
mvn versions:commit  # or versions:revert
```

### Selective
```bash
# Update specific dependency
mvn versions:use-latest-versions -Dincludes=org.junit.jupiter
```
