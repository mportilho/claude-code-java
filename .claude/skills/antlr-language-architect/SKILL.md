---
name: antlr-language-architect
description: ANTLR 4.13.2+ language engineering and grammar architecture patterns. Use when designing, optimizing, testing, or reviewing ANTLR 4 grammars (.g4 files) structure, resolving ALL(*) lookahead issues, or implementing Grammar-Constrained Decoding (GCD) for LLMs.
---

# ANTLR Language Engineer & Architect

Design, optimize, debug, and test ANTLR 4 grammars following high-performance
patterns required by the ALL(*) adaptive algorithm, strict logical separation
of concerns, and lexical resilience.

## When to Use

- Creating or refactoring grammars (`.g4`) for data formats, compilers, or
  LLM decoders (GCD).
- Minimizing parsing inefficiency, "no viable alternative" errors, or
  extreme memory/CPU usage (OOM) caused by SLL falling back to LL.
- Evaluating existing grammars for "code smells" such as grammar loops,
  greedy regex quantifiers, and lexical redundancy.

## Quick Reference

- **Lexer**: Use `-> channel(HIDDEN)`, avoid overlaps, and `ANY : . ;` fallback.
- **Parser**: Root rule with `EOF`, precedence via physical rule order.
- **Walker**: Listeners for large files, Visitors for logical return values.

## Main Content

### 1. Grammar Construction Checklist

- [ ] **[Critical] Separation of Concerns**: Verify if Lexer and Parser are in
  separate files (`Lexer.g4` and `Parser.g4`).
- [ ] **[Critical] Input Consumption**: Ensure the root Parser rule ends with
  the `EOF` terminal to prevent partial parsing.
- [ ] **[High] Lexical Fallback**: Add a final `ANY : . ;` rule in the Lexer to
  safely capture and report unknown characters.
- [ ] **[Medium] Channel Management**: Use `-> channel(HIDDEN)` for tokens like
  whitespace/comments if the original source must be preserved (GCD/Formatting).
- [ ] **[Medium] Token Discarding**: Use `-> skip` if whitespace/comments are
  irrelevant to the application to save memory/CPU.

```antlr
// ✅ GOOD: Root rule ensures full stream consumption
program : statement* EOF ;

// ✅ GOOD: Lexical fallback allows graceful error handling
ANY : . ;
```

### 2. Patterns and "Code Smells" Checklist

- [ ] **[High] Non-Greedy Quantifiers**: Use `.*?` or `.+?` for strings and
  blocks to prevent OOM errors caused by greedy `.*` matching.
- [ ] **[High] Literal Encapsulation**: Replace string literals in the Parser
  (e.g., `'if'`) with explicit Lexer tokens (e.g., `IF`).
- [ ] **[Critical] Left-Recursion Identification**: Ensure all left-recursion
  is "Direct". Eliminate "Indirect Left-Recursion" as it blocks generation.
- [ ] **[High] Explicit Associativity**: Use `<assoc=right>` for operators like
  exponentiation (`^`) or assignment (`=`) that compute right-to-left.

```antlr
// ❌ BAD: Greedy quantifier scans until the *last* quote in the file
STRING : '"' .* '"' ; 

// ✅ GOOD: Non-greedy stops at the *first* quote
STRING : '"' .*? '"' ;

// ✅ GOOD: Explicit associativity for power operators
expr : expr '^' expr # power
     | <assoc=right> expr '=' expr # assignment
```

### 3. Implementation and Performance Checklist

- [ ] **[High] Stream Walker Choice**: Use **Listeners** for large files
  (>10MB) to avoid `StackOverflowError` via the call stack.
- [ ] **[Medium] Logical Walker Choice**: Use **Visitors** only when you need
  to manually control node traversal or return aggregate values.
- [ ] **[Critical] Semantic Predicate Audit**: Ensure predicates `{...}?` are
  not used for syntax decisions that could be handled by pure grammar rules.
- [ ] **[High] Two-Stage Strategy**: Implement SLL-mode first, falling back to
  LL-mode only on prediction errors for maximum throughput.

### 4. Testing and Diagnostics Checklist

- [ ] **[Medium] Ambiguity Profiling**: Run `antlr4-parse --profiler` to identify
  rules that cause excessive lookahead or adaptive backtracking.
- [ ] **[Low] LISP-Tree Validation**: Use TestRig (grun) with `-tree` or `-gui`
  to compare actual syntax trees against expected hierarchical structures.

## Token Optimization

- [ ] Limit evaluation to specific modified .g4 sections unless explicitly
  instructed to perform a full language architecture review.
- [ ] Do not assume Java API validations; focus on the grammar logic.
