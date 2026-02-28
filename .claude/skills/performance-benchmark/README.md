# Performance Benchmark

> Evidence-based performance validation for Java applications using JMH.

## What It Does
The Performance Benchmark skill provides a rigorous workflow for optimizing Java code. It ensures that every performance claim is backed by empirical data from JMH (Java Microbenchmark Harness) and that functional integrity is preserved through comprehensive testing and historical logging.

## When to Use
- **Trigger phrases:** "optimize this hot path", "check for performance regressions", "improve throughput", "reduce latency", "validate performance in this PR".
- **Scenarios:** Refactoring core logic, upgrading libraries that affect performance, or resolving performance bottlenecks identified in production.

## Key Concepts
- **Evidence-Based Decision:** Decisions to accept or discard code changes are based on comparative benchmarks (`before` vs. `after`).
- **JMH Mandatory:** Only JMH-based benchmarks are accepted for performance validation.
- **Functional Safety First:** Functional tests must be verified before any performance measurements are taken.
- **Historical Audit Trail:** All experiments are recorded in an append-only `performance-history.md` file within the relevant module.

## Example Usage
**User:** "I want to optimize the `calculateTax` method in the `BillingService`. It seems to be a bottleneck."

**Claude:** "I will apply the **Performance Benchmark** skill. 
1. I'll start by mapping the current functional tests and creating a JMH baseline for `calculateTax`.
2. Then, I'll implement the optimization and verify it against functional tests and edge cases.
3. Finally, I'll run the same benchmark to compare results and record the experiment in `docs/perf/performance-history.md`."

## Related Skills
- **performance-smell-detection:** For identifying potential bottlenecks before applying the quality gate.
- **test-quality:** For ensuring high-quality functional tests during the verification phase.
- **java-code-review:** For general code quality checks alongside performance optimization.

## References
- [JMH (Java Microbenchmark Harness)](https://github.com/openjdk/jmh)
- [Project JMH Playbook](references/jmh-playbook.md)
- [Performance History Template](references/performance-history-template.md)
