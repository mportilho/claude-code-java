# ANTLR Language Engineer & Architect

> Padrões, arquitetura e boas práticas para gramáticas ANTLR 4 de alto desempenho

## What It Does
Este skill fornece heurísticas estruturadas para a criação, análise, teste e otimização de gramáticas e interpretadores/compiladores baseados no ANTLR 4. Aplica os melhores padrões modernos da versão 4 do ANTLR focando no algoritmo adaptativo ALL(*).

## When to Use
- Criar gramáticas (`.g4`) para nova linguagem, analisador ou parser.
- Encontrar ambiguidades ou resolver problemas de lookahead no parsing.
- Guiar a implementação de Visitors ou Listeners para percorrer uma Parse Tree.
- Avaliar gramáticas existentes procurando code smells de linguagens.
- Validar se o teste e manuseio de erros no ANTLR está feito da forma ótima.

## Key Concepts
- **Algoritmo ALL(*)**: Análise adaptativa em tempo de execução que evoluiu com ANTLR 4.
- **LL() vs SLL()**: Parsing de dois estágios otimiza performance na maioria dos casos.
- **Separação Léxica**: Tokens (Lexer) separados das regras (Parser) garantem manutenibilidade, usando `channel(HIDDEN)` para metadados (como espaços e comentários).
- **Visitor e Listener**: Duas formas de percorrer a árvore com trade-offs em flexibilidade vs estabilidade de stack.

## Example Usage
> "We need to parse a custom query language. Can you write the grammar for it?"

*Claude will use the templates and rules in this skill to correctly split into QueryLexer.g4 and QueryParser.g4 appropriately, applying best practices such as rule ANY and EOF token consumption.*

> "The AST visitor is throwing StackOverflowError on deeply nested expressions"

*Claude will reference this skill to suggest applying Listeners instead of Visitors for deeply recursive components, or restructuring the grammar structure.*

## Related Skills
- [architecture-review](../architecture-review/) - Macro review of how parsers integrate globally in Java.
- [test-quality](../test-quality/) - Best practices in isolation tests (JUnit 5 / AssertJ) applicable to grammar parser testing.
