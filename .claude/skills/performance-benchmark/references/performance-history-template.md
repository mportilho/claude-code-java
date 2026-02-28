# Template de Registro (Append-Only)

> Arquivo de destino obrigatório: `<módulo>/docs/perf/performance-history.md`
> Em multi-módulo, usar sempre o submódulo alvo (ex.: `expression-evaluator/docs/perf/performance-history.md`).

```md
## Experimento PERF-XXX
- Data: YYYY-MM-DD
- Objetivo:
- Baseline commit/estado:
- Commit/estado testado:

### Hipótese
1.
2.

### Mudanças aplicadas
- `arquivo:linha`:
- `arquivo:linha`:

### Resumo de medição
- JVM:
- JMH:
- Benchmark class (src/test/java):
- Package de benchmark:
- Cenário:
- Parâmetros-chave:

### Resultado
| Benchmark | Before (ns/op) | After (ns/op) | Delta (ns/op) | Melhoria (%) |
|---|---:|---:|---:|---:|
| ... | ... | ... | ... | ... |

### Decisão
- [ ] Aceitar
- [ ] Ajustar
- [ ] Descartar

### Leitura técnica
1.
2.

### Atividades executadas
1. baseline funcional/JMH - sucesso/falha
2. validação pós-mudança - sucesso/falha
3. registro no histórico - sucesso/falha

### Riscos residuais
1.
2.
```

## Convenções
- Manter histórico append-only.
- Não sobrescrever resultados anteriores.
- Sempre registrar `Melhoria (%)` baseada em comparação de `ns/op`.
