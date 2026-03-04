# ANTLR Java Integration

> Patterns for scaling, embedding, and optimizing ANTLR 4.13+ within Java applications.

## What It Does

Provides an actionable checklist and best practices for integrating ANTLR 4.13+ generated parsers in Java. It specifically targets common pitfalls found in production systems, such as `OutOfMemoryError` when parsing huge files, memory leaks caused by unbounded ATN (DFA) caches, and thread-safety violations during parsing and template generation.

## When to Use

- "Review this ANTLR parser execution code for memory leaks."
- "How do I process a 5GB file with ANTLR in Java?"
- "We are getting OutOfMemoryError in our parser."
- "Integrate this ANTLR grammar into our Spring Boot 4 application safely."
- "Write a Thread-safe parser execution block using this ANTLR grammar."

## Key Concepts

### ATN Cache Leaks

The ALL(*) parsing algorithm caches parsing decisions to speed up future predictions. Without manual cache clearing (`clearDFA()`), this structure can exhaust JVM memory in long-running services.

### Unbuffered Parsing

Using `UnbufferedCharStream` and `UnbufferedTokenStream` prevents ANTLR from loading massive input files entirely into RAM.

### Strict Thread Isolation

Lexer, Parser, and Traverser implementations are stateful. Their instances must be completely isolated per processing thread.

## Example Usage

### 1. Modern Error Handling

The default ANTLR error listener prints warnings to the console without interrupting execution, which is dangerous for production. Instead, create a custom listener to capture or fail-fast.

```java
public class SyntaxErrorCollector extends BaseErrorListener {
    private final List<String> errors = new ArrayList<>();

    @Override
    public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol,
                            int line, int charPositionInLine, String msg, RecognitionException e) {
        errors.add(String.format("Error at %d:%d: %s", line, charPositionInLine, msg));
    }
    public List<String> getErrors() { return errors; }
}

// Applying it safely:
MyLexer lexer = new MyLexer(CharStreams.fromString("input"));
lexer.removeErrorListeners(); // Remove default console listener
lexer.addErrorListener(new SyntaxErrorCollector());

MyParser parser = new MyParser(new CommonTokenStream(lexer));
parser.removeErrorListeners(); 
SyntaxErrorCollector parserErrors = new SyntaxErrorCollector();
parser.addErrorListener(parserErrors);
```

### 2. Processing Giant Files (Anti OutOfMemoryError)

```java
// ❌ BAD: Shared parser instance or buffered parsing for massive files
public static Parser parser = new MyParser(new CommonTokenStream(new MyLexer(CharStreams.fromFileName("huge.log"))));

// ✅ GOOD: Isolated, unbuffered execution for large files
InputStream stream = new FileInputStream("huge.log");
UnbufferedCharStream input = new UnbufferedCharStream(stream);
MyLexer lexer = new MyLexer(input);
lexer.setTokenFactory(new CommonTokenFactory(true)); 

UnbufferedTokenStream tokens = new UnbufferedTokenStream(lexer);
MyParser parser = new MyParser(tokens);
parser.setBuildParseTree(false); // Disable tree building to save memory
```

## Related Skills

- [antlr-language-architect](../antlr-language-architect/) - For grammar design (`.g4`), lexer/parser separation, and optimization.
- [performance-smell-detection](../performance-smell-detection/) - For general Java performance concepts.

## References

- The Definitive ANTLR 4 Reference - Terence Parr
- Adaptive LL(*) Parsing: The Power of Dynamic Analysis - ANTLR
- Optimizing ANTLR Memory Footprint
