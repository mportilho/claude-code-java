---
name: antlr-java-integration
description: ANTLR 4.13+ Java integration, memory management, and thread-safety patterns. Use when integrating ANTLR parsers in Java or reviewing ANTLR Java execution code.
---

# ANTLR Java Integration

Guide for implementing, optimizing, and scaling ANTLR 4.13+ parsers within Java applications. Focuses on memory management, thread-safety, and runtime performance.

## When to Use

- Integrating an ANTLR 4.13+ `.g4` generated parser into a Java project
- Reviewing Java code that invokes ANTLR Lexers or Parsers
- Debugging `OutOfMemoryError` or memory leaks in ANTLR parsing
- Optimizing ANTLR parsing performance in Java (e.g., handling very large files)
- Implementing multithreaded parsing or StringTemplate 4 (ST4) generation

## Quick Reference

| Scenario | Recommended Pattern |
|---|---|
| **Memory Leaks** | Call `interpreter.clearDFA()` periodically in long-running processes to clear ATN cache. |
| **Large Files (GBs)** | Use `UnbufferedCharStream`, `UnbufferedTokenStream`, and disable parse tree building. |
| **Thread-Safety** | Instantiate new Lexer, Parser, and Traverser per thread. `STGroupFile` is not thread-safe. |
| **Grammar Design** | Use `#` labels for specific visitor methods and always include `EOF` in start rule. |
| **Ambiguity** | Order Lexer rules from most specific (keywords) to general (identifiers). |
| **Predicates** | Minimize complex Java logic in `{...}?` predicates as they run during prediction. |
| **Cold-Start** | Run parsing on dummy data at startup to populate the ATN DFA cache. |

## Main Content

### 1. Memory Management & Scalability

- **Handling Giant Files**: By default, ANTLR buffers the entire input and builds a complete Parse Tree in memory. This causes `OutOfMemoryError` on large data streams (e.g., GB-sized logs).
  - Use `UnbufferedCharStream` instead of standard `CharStreams`.
  - Use `UnbufferedTokenStream` instead of `CommonTokenStream`.
  - Disable tree building: `parser.setBuildParseTree(false)`.
  - Process via embedded grammar actions or immediate reduction, bypassing in-memory ASTs.
- **ATN Cache Management (DFA Leaks)**: The ALL(*) algorithm builds a DFA cache dynamically. In long-running applications (like web servers) parsing diverse inputs, this cache grows indefinitely, causing memory leaks.
  - Mitigation: Monitor memory and periodically clear the cache via `lexer.getInterpreter().clearDFA()` and `parser.getInterpreter().clearDFA()`.

### 2. Thread-Safety and Concurrency

- **Isolate Parser Instances**: Parsers, Lexers, and TokenStreams are inherently stateful and **NOT** thread-safe. Never share them across threads. Create new instances per request or thread.
- **Stateless Listeners/Visitors**: When using a shared or singleton Listener/Visitor, do not store parse state in instance fields.
- **StringTemplate 4 (ST4) Thread-Safety**: If generating code using ST4 alongside ANTLR, be aware that `STGroupFile` is not thread-safe during template loading.
  - Cache `CompiledST` objects securely.
  - Use initialization-on-demand or synchronize the initial STGroup loading.
  - In modern Java (21-25+), avoid `ThreadLocal`. Use `ScopedValue` for safe and efficient sharing of ST instances, which is highly optimized for Virtual Threads.

### 3. Tree Traversal & Evaluation

- **Listener Pattern**: Ideal for tasks that do not return values (like linting, symbol table populating, or formatting). It is event-driven and immune to stack overflows for deeply nested ASTs.
- **Visitor Pattern**: Best for interpreters and AST-to-AST translation, where evaluating a node requires returning a value to its parent. Require `-visitor` flag during standard generation.

### 4. Integration & First-Run Performance

- **Two-Stage Parsing Strategy**: To maximize speed while maintaining error recovery, try parsing in `SLL` mode first using `BailErrorStrategy`. If it fails, fallback to standard `LL` mode using `DefaultErrorStrategy`.
- **Warm-Up Phase**: The first time an ALL(*) parser runs, it lazily builds the DFA transition tables. This "cold start" can be up to 10x slower than subsequent runs.
  - For latency-sensitive systems, run a set of representative parse operations during application startup to warm up the ATN cache.
- **Custom Error Handling**: Avoid default console error reporting in production. Always remove default listeners (`removeErrorListeners()`) and provide a custom `ANTLRErrorListener` to collect errors or log them to standard APM tools.
- **Indirect Recursion**: ANTLR 4 does not support indirect left-recursion (e.g., `a: b; b: a;`). If encountered, it will fail during the Java code generation/build phase.

### 5. Grammar-to-Java Optimization

- **Start Rule EOF**: Always end the main entry rule with `EOF`. Without it, the parser may stop early on a valid prefix and ignore trailing junk, leading to hard-to-debug integration issues.
- **Visitor Labeling**: Use `#` labels on alternatives for complex rules. This forces ANTLR to generate specific visitor/listener methods (e.g., `visitAddition`) instead of a single generic method, making the Java implementation cleaner.
- **Lexer Rule Order**: ANTLR Lexers are "greedy" and match rules in declaration order. Define specific keywords **above** general identifiers to avoid incorrect tokenization in Java.
- **Predicates Complexity**: Java code inside semantic predicates (e.g., `{isType(ID)}?`) runs frequently during the ATN prediction phase. Keep this logic lean to avoid performance degradation.

### 5. Modern Custom Error Handling

Relying on ANTLR's default console output is a bad practice in production. You must collect errors structuredly or fail-fast.

The usefulness of a parser is heavily dependent on the quality of its error recovery strategies.

| Error Strategy / Listener | Behavior | Recommended Use Case |
|---|---|---|
| **DefaultErrorStrategy** | Attempts to recover by inserting/removing tokens. | IDEs and text editors with real-time feedback. |
| **BailErrorStrategy** | Fails immediately, throwing `ParseCancellationException`. | CLI compilers, CI, and Batch processing. |
| **DescriptiveErrorListener** | Generates detailed messages with visual context. Implements `BaseErrorListener`. | Tools aimed at end-users and learning. |
| **ThrowingErrorListener** | Converts syntax errors into runtime exceptions. | Integration with unit testing frameworks and autonomous agents. |

Use this modern `BaseErrorListener` (collector) pattern to catch errors without polluting standard output, which is especially useful for APIs and Language Servers:

```java
import org.antlr.v4.runtime.BaseErrorListener;
import org.antlr.v4.runtime.RecognitionException;
import org.antlr.v4.runtime.Recognizer;
import java.util.ArrayList;
import java.util.List;

public class SyntaxErrorCollector extends BaseErrorListener {
    private final List<String> errors = new ArrayList<>();

    @Override
    public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol,
                            int line, int charPositionInLine,
                            String msg, RecognitionException e) {
        // Collect formatted error for API response or logging
        errors.add(String.format("Syntax error at line %d:%d - %s", line, charPositionInLine, msg));
    }

    public List<String> getErrors() {
        return errors;
    }
    
    public boolean hasErrors() {
        return !errors.isEmpty();
    }
}
```

**Fail-Fast Error Handling (Throwing Exceptions):**
If you need validation to stop immediately on the first error (e.g., CI pipelines or strict compilation), throw a `ParseCancellationException`.

```java
import org.antlr.v4.runtime.misc.ParseCancellationException;

public class ThrowingErrorListener extends BaseErrorListener {
    public static final ThrowingErrorListener INSTANCE = new ThrowingErrorListener();

    @Override
    public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol, 
                            int line, int charPositionInLine, String msg, RecognitionException e) {
        throw new ParseCancellationException("line " + line + ":" + charPositionInLine + " " + msg);
    }
}
```

**Registering the Listener on Lexer and Parser:**
Make sure to remove the default error listeners before attaching your custom listener, otherwise errors will still be printed to the console:

```java
public static String parse(String text) throws ParseCancellationException {
    MyLexer lexer = new MyLexer(CharStreams.fromString(text));
    lexer.removeErrorListeners(); // Remove default ConsoleErrorListener
    lexer.addErrorListener(ThrowingErrorListener.INSTANCE);

    CommonTokenStream tokens = new CommonTokenStream(lexer);
    
    MyParser parser = new MyParser(tokens);
    parser.removeErrorListeners(); // Remove default ConsoleErrorListener
    parser.addErrorListener(ThrowingErrorListener.INSTANCE);

    ParserRuleContext tree = parser.expr();
    MyParseRules extractor = new MyParseRules();

    return extractor.visit(tree); // Execute traversal
}
```

### 7. Severity & Risk Matrix

| Severity | Issue | Impact | Mitigation |
|---|---|---|---|
| **Critical** | Indirect Left-Recursion | Build fails. Code generation is impossible. | Rewrite grammar to remove cycles or inline rules. |
| **High** | OOM on Giant Files | JVM crash. Builds/Parses massive trees in memory. | Use unbuffered streams and `parser.setBuildParseTree(false)`. |
| **High** | ATN/DFA Cache Leak | Progressive RAM exhaustion in servers. | Periodically call `clearDFA()` on interpreters. |
| **Medium** | Missing EOF in Start Rule | Silent failures on invalid input suffixes. | Append `EOF` to the start rule. |
| **Medium** | Slow Semantic Predicates | Parsing performance degrades during prediction. | Optimize Java logic inside `{...}?` predicates. |
| **Low** | Missing Rule Labels | Harder to maintain generic Visitors/Listeners. | Use `#` labels for specific method generation. |

## Anti-patterns

❌ **Avoid**:

```java
// 1. Loading giant files fully into memory
CharStream input = CharStreams.fromFileName("multi-gigabyte-log.txt"); // Might OutOfMemory Error
MyLexer lexer = new MyLexer(input);
CommonTokenStream tokens = new CommonTokenStream(lexer); 
MyParser parser = new MyParser(tokens);
ParseTree tree = parser.root(); // Builds massive tree in memory

// 2. Sharing stateful components across threads
// BAD: defined as static or Spring Boot 4 @Bean(scope="singleton")
@Bean
public MyParser sharedParser() {
    return new MyParser(null);
}

// 3. Leaving default error listeners in production
MyParser parser = new MyParser(tokens);
// BAD: Default listener prints to System.err quietly
parser.parse();
```

✅ **Prefer**:

```java
// 1. Unbuffered parsing for massive files
InputStream is = new FileInputStream("multi-gigabyte-log.txt");
UnbufferedCharStream input = new UnbufferedCharStream(is);
MyLexer lexer = new MyLexer(input);
lexer.setTokenFactory(new CommonTokenFactory(true));
UnbufferedTokenStream tokens = new UnbufferedTokenStream(lexer);
MyParser parser = new MyParser(tokens);
parser.setBuildParseTree(false); // Prevents tree memory explosion

// 2. Thread-local or Request-scoped parser instantiation
public CompilationResult parseCode(String code) {
    MyLexer lexer = new MyLexer(CharStreams.fromString(code));
    MyParser parser = new MyParser(new CommonTokenStream(lexer));
    // Thread-safe!
}

// 3. Custom Error listener integration
MyParser parser = new MyParser(tokens);
parser.removeErrorListeners(); // Remove console noise
SyntaxErrorCollector collector = new SyntaxErrorCollector();
parser.addErrorListener(collector);
```

## Audit Checklist

When reviewing or writing ANTLR Java integration code, verify the following:

- [ ] **[High] Thread-Safety**: Lexer, Parser, and Traverser (Visitor/Listener) are instantiated per thread or request?
- [ ] **[High] Large File Capability**: For file processing, is it using `UnbufferedCharStream` and `UnbufferedTokenStream` to avoid `OutOfMemoryError`?
- [ ] **[High] DFA Memory Leak Mitigations**: In long-running processes (e.g. web servers), is `clearDFA()` called on Lexer and Parser interpreters to free ATN cache?
- [ ] **[Medium] Error Handling**: `removeErrorListeners()` is called and a custom, non-printing `ANTLRErrorListener` is attached instead of just logging to `System.err`?
- [ ] **[Medium] StringTemplate Security**: Are `STGroup` and `CompiledST` instances safely shared using `ScopedValue` or initialization synchronization?
- [ ] **[Low] Tree Building Scope**: Is `parser.setBuildParseTree(false)` called when processing massive inputs where an AST manipulation is unnecessary?
- [ ] **[Low] Performance Warmup**: Is a dummy parse running at startup to warm up the ALL(*) DFA cache in latency-critical scenarios?
- [ ] **[Low] Start Rule Integrity**: Does the entry rule end with `EOF` to ensure full input consumption?
- [ ] **[Low] Rule Labeling**: Are `#` labels used in complex rules to simplify the Java Visitor/Listener?
- [ ] **[Low] Test Isolation**: Are Lexer and Parser components tested independently (e.g. mocking token streams via `ListTokenSource`)?

## Token Optimization

1. **Exit Early**: If the project does not parse files larger than ~10MB, skip large file unbuffered input checks to save tokens.
2. **Commands**: Use the following tool to quickly identify where ANTLR is executed in the codebase:

   ```bash
   rg -t java "CharStreams\.from|UnbufferedCharStream|new .*Parser\(|clearDFA"
   ```

3. Prioritize memory and thread-safety checks when analyzing `Lexer` and `Parser` instantiations. Skip `.g4` grammar design checks (redirect those to `antlr-language-architect`).
