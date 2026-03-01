---
name: antlr-language-architect
description: ANTLR 4 language engineering and architecture patterns. Use when designing, optimizing, testing, or reviewing ANTLR 4 grammars (.g4 files) and parsers.
---

# ANTLR Language Engineer & Architect

Projetar, otimizar, depurar e testar gramáticas ANTLR 4 seguindo padrões de alto desempenho, separação de preocupações e robustez na captura de erros.

## When to Use
- Criar ou modificar arquivos `.g4`
- Otimizar gramáticas ANTLR4 existentes
- Implementar Visitors ou Listeners para processamento de árvores
- Escrever testes para parsers ANTLR

## Quick Reference
| Componente | Padrão Recomendado |
|---|---|
| Léxico | Separar em `Lexer.g4`, usar `-> channel(HIDDEN)` para metadados, terminar com `ANY : . ;` |
| Parser | Separar em `Parser.g4`, terminar regra inicial com `EOF`, rotular alternativas (`# label`) |
| Runtime | Modo SLL inicial com `BailErrorStrategy`, fallback para LL com `DefaultErrorStrategy` |
| Travessia | `Visitor` para transformação/AST, `Listener` para linting (evita stack overflow) |

## Main Content

### 1. Protocolo de Design de Gramática
- **Separação de Arquivos**: Sempre separar a gramática em Lexer (`Lexer.g4` com tokens em maiúsculas) e Parser (`Parser.g4` com regras em minúsculas) para evitar ambiguidades.
- **Consumo Total (EOF)**: A regra inicial do parser deve obrigatoriamente terminar com o token `EOF`.
- **Robustez do Léxico**: Adicionar a regra genérica no final: `ANY : . ;`. Isso garante um token de erro explícito.
- **Gestão de Metadados**: Utilizar `-> channel(HIDDEN)` para espaços/comentários em vez de `-> skip`.
- **Rotulagem de Alternativas**: Rótulos (`# addExpr`) obrigatórios em alternativas de regras complexas.

### 2. Padrões de Expressões e Precedência
- **Recursão à Esquerda**: Utilizar recursão direta nativa, ordenando alternativas da maior para a menor precedência.
- **Associatividade**: Usar `<assoc=right>` explicitamente para operadores associativos à direita (ex: exponenciação).
- **Inlining**: Integrar sub-produções de identificadores em chamadas de função para reduzir lookahead no modo SLL.

### 3. Heurísticas de Análise Estática e Otimizações
Verificar gramáticas buscando "smells":
- **Missing ANY Rule**: Ausência de captura genérica de léxico no final do arquivo.
- **Optional Semicolon Ambiguity**: Terminadores opcionais sem delimitação clara.
- **Recursive Indirect Loops**: Ciclos (A -> B -> A) não solucionáveis pelo ANTLR4.
- **Tokens Literais no Parser**: Uso direto de strings (ex: 'while') no parser. Eles devem ser migrados para declarações no léxico.

**Técnicas de Otimização no Nível da Gramática:**
| Técnica | Descrição |
|---|---|
| **Parsing de 2 Estágios** | Prioriza o modo SLL antes de recorrer ao LL completo (reduz drásticamente custos cpu/ram). |
| **Inlining de Funções** | Elimina regras aninhadas substituindo por tokens mais robustos gerenciados pelo léxico. |
| **Fatoração à Esquerda** | Mescla alternativas que possuem prefixos idênticos para evitar redundância na árvore de parsing limitando o custo de lookahead. |
| **Remoção de Parênteses** | Elimina regras redundantes de agrupamento (e sub-árvores inúteis) se elas já são cobertas logicamente por expressões ordenadas por precedência. |

### 4. Estratégia de Runtime e Travessia
- **Estratégia de Dois Estágios**: Tentar SLL com `BailErrorStrategy`, recorrer a LL com `DefaultErrorStrategy` em caso de falha.
- **Visitor vs. Listener**:
  - `Visitor`: Controle manual e retorno de valores, ideal para interpretadores/AST.
  - `Listener`: Orientado a eventos, progressão automática, ideal para linting/auditoria (seguro contra `Stack Overflow`).

### 5. Protocolo de Testes e Erros
A utilidade de um analisador sintático é frequentemente medida pela qualidade de suas estratégias de recuperação de erros.

| Estratégia de Erro / Listener | Comportamento | Cenário de Uso Recomendado |
|---|---|---|
| **DefaultErrorStrategy** | Tenta recuperar inserindo/removendo tokens. | IDEs e editores de texto com feedback em tempo real. |
| **BailErrorStrategy** | Interrompe imediatamente, lançando exceção (`ParseCancellationException`). | Compiladores CLI, CI e processamento Batch. |
| **DescriptiveErrorListener** | Gera mensagens detalhadas com contexto visual. Implementação do `BaseErrorListener`. | Ferramentas voltadas para usuários finais e aprendizado. |
| **ThrowingErrorListener** | Converte erros de sintaxe em check-exceptions de tempo de execução. | Integração com frameworks de teste unitário e agentes autônomos. |

**Exemplo de Implementação: ThrowingErrorListener (Fail-Fast)**
Para interromper a execução assim que um erro de léxico ou parser ocorrer — ideal para builds strict ou fail-fast —, implemente um listener customizado que lance `ParseCancellationException`.

```java
import org.antlr.v4.runtime.BaseErrorListener;
import org.antlr.v4.runtime.RecognitionException;
import org.antlr.v4.runtime.Recognizer;
import org.antlr.v4.runtime.misc.ParseCancellationException;

public class ThrowingErrorListener extends BaseErrorListener {
    public static final ThrowingErrorListener INSTANCE = new ThrowingErrorListener();

    @Override
    public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol, int line, int charPositionInLine, String msg, RecognitionException e)
        throws ParseCancellationException {
        // Lança ParseCancellationException em vez de RecognitionException
        throw new ParseCancellationException("line " + line + ":" + charPositionInLine + " " + msg);
    }
}
```

**Registro do Listener no Lexer e Parser:**
Certifique-se de remover os listeners de erro padrão antes de acoplar sua customização:

```java
public static String parse(String text) throws ParseCancellationException {
    MyLexer lexer = new MyLexer(CharStreams.fromString(text));
    lexer.removeErrorListeners(); // Remove o ConsoleErrorListener padrão
    lexer.addErrorListener(ThrowingErrorListener.INSTANCE);

    CommonTokenStream tokens = new CommonTokenStream(lexer);
    
    MyParser parser = new MyParser(tokens);
    parser.removeErrorListeners(); // Remove o ConsoleErrorListener padrão
    parser.addErrorListener(ThrowingErrorListener.INSTANCE);

    ParserRuleContext tree = parser.expr();
    MyParseRules extractor = new MyParseRules();

    return extractor.visit(tree);
}
```

- **Isolamento de Testes**: Testar Lexer e Parser separadamente (Injete sequências como mock ao parser usando `ListTokenSource`).
- **Error Listener Contextual**: Sempre que possível, intercepte `RecognitionException` sobrescrevendo `syntaxError()` para injetar marcadores (ex: `> <`) apontando o token problemático no log/console.

## Token Optimization
- Limite a verificação da gramática aos arquivos `.g4` específicos ou trechos afetados.
- Se não for pedido especificamente para mudar a gramática ou resolver problemas de parsing, não aplique validadores gramaticais completos.
