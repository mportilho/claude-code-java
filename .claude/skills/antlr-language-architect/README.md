# ANTLR Language Engineer & Architect

> Patterns, architecture, and best practices for high-performance ANTLR 4
> grammars

## What It Does

This skill provides structured heuristics for creating, analyzing, testing, and
optimizing logical grammars and interpreters / compilers based on ANTLR 4.13.2+.
It focuses predominantly on enabling the full power of the **ALL(*) adaptive
algorithm** by limiting SLL fallback to LL, resolving syntactic performance
bottlenecks, and applying constraints to LLM *Grammar-Constrained Decoding*
(GCD).

## When to Use

- **"Help me optimize this parser..."**: Minimize inefficiency, ambiguities
  (*"no viable alternative"*), or extreme memory/CPU consumption (`OOM`).
- **"Can you write an ANTLR grammar for..."**: Create or refactor grammars
  (`.g4`) for structured data, abstract languages, or JSON inference decoders
  for LLMs without "reasoning gaps".
- **"I need to extract data from this large log using ANTLR..."**: Scale the
  lexical architecture using hidden channels and design flexible tree
  strategies using *Listeners* to avoid degrading the Java call stack.
- **"Evaluate if this .g4 is correct"**: Evaluate text files and logical
  grammars to avoid *loops*, intrusive semantic predicates, and regex
  *greediness* (Quantifiers).

## Key Concepts

- **ALL(*) Algorithm (Adaptive LL(*))**: Dynamic top-down analysis capable of
  infinite lookahead at runtime by building DFAs in memory.
- **Two-Stage Parsing (SLL/LL)**: Central tactic of the ANTLR4 ecosystem:
  attempting to derive based strictly on a weaker context (*Strong-LL* / fast),
  falling back to the complex adaptive backtracking environment (LL) if - and
  only if - lexical/syntactic errors occur in SLL.
- **Grammar-Constrained Decoding (GCD)**: LLM models often need to format their
  output. Filters and restricted decoding fundamentally depend on lexical
  modeling and ignore global arbitrary Java class verification.
- **Hidden Lexical Channeling**: ANTLR v4 decouples white spaces and comments
  from the syntax flow using `channel(HIDDEN)`.
- **Listener vs Visitor**: Critical decision limiting processing strategies
  (Stack management).

## Example Usage

> "The parser is consuming GBs of heap memory and causing OOM when processing a
> 50MB file. Can you fix the performance issue?"

*Claude will use the heuristics here to review quantifier greediness
(`.*` to `.*?`), evaluate if tokens are recursively redundant, and propose
migrating logic from massive AST Visitor allocations to event-driven Stream
Listeners.*

## Related Skills

- [architecture-review](../architecture-review/) - Macro review of how parsers
  integrate globally in Java.
- [test-quality](../test-quality/) - Best practices in isolation tests
  (JUnit 5 / AssertJ) applicable to grammar parser testing.
- [antlr-java-integration](../antlr-java-integration/) - Java-specific
  implementation details (ErrorListeners, Walker strategies implementation).

## References

- [Melhores Práticas ANTLR Gramáticas .g4.md](../../references/Melhores%20Práticas%20ANTLR%20Gramáticas%20.g4.md)
