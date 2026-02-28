---
name: performance-benchmark
description: Conduct performance and quality analysis for Java changes with a focus on evidence-based decision making. Use for code optimization, suspected latency/throughput regressions, hot path refactoring, or when validating changes using functional tests and JMH benchmarks (before/after) with historical logging.
---

# Performance Benchmark (JMH)

Ensure Java performance improvements are safe and validated through rigorous JMH benchmarking and functional testing. This skill enforces a data-driven "Accept/Adjust/Discard" decision workflow with mandatory historical records.

## When to Use
- Optimizing "hot paths" or critical business logic.
- Investigating suspected performance regressions (latency/throughput).
- Refactoring high-traffic components or core libraries.
- Validating performance claims with empirical evidence.

## Quick Reference: The Quality Gate Workflow

| Step | Action | Outcome |
|------|--------|---------|
| **1. Baseline** | Map hot path and measure with JMH | `before` metrics and functional tests |
| **2. Implement** | Minimal change for maximum impact | Optimized code |
| **3. Verify** | Run functional tests + Edge cases | Behavioral confidence |
| **4. Validate** | Re-run JMH under identical conditions | `after` metrics and delta (%) |
| **5. Decide** | Accept, Adjust, or Discard | Final decision based on evidence |
| **6. Record** | Update `performance-history.md` | Append-only audit trail |

## Operational Workflow

### 1. Scope and Hypothesis
- Identify the hot path, load type, and regression risks.
- Define primary metrics (e.g., `ns/op` for latency, `ops/s` for throughput).
- Establish the technical baseline (specific commit, branch, or tree state).

### 2. Measure Baseline (Before)
- Ensure the module has **JMH enabled**. If missing, configuration is mandatory before proceeding.
- Create/adjust benchmarks in `src/test/java` within a dedicated performance package (e.g., `...perf.jmh...`).
- Run JMH with fixed parameters and record results.
- **Mandatory:** Functional tests must pass *before* benchmarking to ensure the baseline is correct.

### 3. Minimal Secure Implementation
- Apply the smallest change necessary to address the hotspot.
- Preserve public semantics and prioritize simplicity in the hot path.
- Avoid unrelated refactorings in the same delivery.

### 4. Functional Safety
- Update or add unit/integration tests covering:
    - Nominal behavior.
    - Optimization edge cases.
    - Compatibility invariants.
- **Pre-check:** Ensure high-confidence coverage of the hot path *before* applying the optimization.
- **Post-check:** Re-run all tests to ensure no behavioral regressions.

### 5. Measure "After"
- Execute the same JMH benchmarks using **identical parameters**.
- Capture score and error margins.
- Calculate absolute and percentage deltas:
  `Improvement (%) = ((before_ns_op - after_ns_op) / before_ns_op) * 100`

### 6. Evidence-Based Decision
- **Accept:** Significant gain or negligible regression with clear structural benefits.
- **Adjust:** Partial gain with significant regressions or residual functional risks.
- **Discard:** Regression without justifiable counter-benefits.
- Explicitly state any inferences vs. hard data.

### 7. Historical Record
- Create or update `docs/perf/performance-history.md` in the **target module directory**.
- In multi-module projects, use the specific submodule path (e.g., `expression-evaluator/docs/perf/performance-history.md`).
- Append the experiment: Hypothesis, changes, measurement summary, results, decision, and lessons learned.

## Mandatory Quality Standards

- **Functional tests must pass** with high confidence before optimization.
- **JMH is mandatory** for performance validation; ad-hoc timers (e.g., `System.currentTimeMillis()`) are prohibited.
- Benchmarks must reside in `src/test/java` in a dedicated `perf.jmh` package.
- **Historical logging** in `performance-history.md` is required for every significant change.
- Comparison tables must include `before ns/op`, `after ns/op`, and `Improvement (%)`.

## Implementation Patterns

### ✅ GOOD: Using JMH for Evidence
```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class StringConcatBenchmark {
    @Benchmark
    public String testStringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) sb.append(i);
        return sb.toString();
    }
}
```

### ❌ BAD: Ad-hoc Timing (Low Confidence)
```java
// DO NOT DO THIS
long start = System.nanoTime();
runOptimization();
long end = System.nanoTime();
System.out.println("Took: " + (end - start));
```

## Token Optimization

1. **Verify JMH first:** Use `mvn test-compile` to ensure benchmarks are valid before running them.
2. **Shorten warmups for quick feedback:** Use `-wi 2 -i 3 -f 1` for initial validation, but use standard parameters for final decisions.
3. **Identify hot paths early:** Use `grep_search` to find high-frequency loops or complex logic before suggesting optimizations.
4. **Early Exit:** If functional tests fail, do not proceed to benchmarking.

## Anti-Patterns
- **Guessing performance:** Making claims without comparative benchmarks.
- **Benchmarking without JMH:** Using unreliable ad-hoc tools.
- **Optimization before validation:** Implementing "fixes" without existing functional test coverage.
- **Polluting the codebase:** Placing benchmarks in `src/main/java` or mixed with unit tests.
- **Ignoring noise:** Accepting results with high error margins or environmental interference.

## References
- **JMH Playbook:** `references/jmh-playbook.md` (Commands and benchmark design).
- **History Template:** `references/performance-history-template.md` (Format for `performance-history.md`).
- **Checklist:** `references/quality-activities-checklist.md` (Step-by-step execution guide).
