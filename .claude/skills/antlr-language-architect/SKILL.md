---
name: antlr-language-architect
description: ANTLR 4 language engineering and grammar architecture patterns. Use when designing, optimizing, testing, or reviewing ANTLR 4 grammars (.g4 files) structure.
---

# ANTLR Language Engineer & Architect

Projetar, otimizar, depurar e testar gramáticas ANTLR 4 seguindo padrões de alto desempenho, separação de preocupações e robustez na captura de erros.

## When to Use
- Criar ou modificar arquivos `.g4`
- Otimizar gramáticas ANTLR4 existentes
- Resolver problemas de lookahead ou ambiguidades sintáticas

## Quick Reference
| Componente | Padrão Recomendado |
|---|---|
| Léxico | Separar em `Lexer.g4`, usar `-> channel(HIDDEN)` para metadados, terminar com `ANY : . ;` |
| Parser | Separar em `Parser.g4`, terminar regra inicial com `EOF`, rotular alternativas (`# label`) |

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



## Token Optimization
- Limite a verificação da gramática aos arquivos `.g4` específicos ou trechos afetados.
- Se não for pedido especificamente para mudar a gramática ou resolver problemas de parsing, não aplique validadores gramaticais completos.
