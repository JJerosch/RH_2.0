# Convenção de Commits

Objetivo: o `git log` vira um **índice navegável** do projeto. Daqui a meses, lendo o histórico, você entende *o que* foi feito e *por quê* — e acha a nota do cofre que explica.

## Formato: Conventional Commits + referência ao cofre

```
<tipo>(<escopo>): <resumo no imperativo, minúsculo>

<corpo: o que e por quê, não o como>

Ref: [[nota do cofre]] · <ADR ou Qn quando aplicável>
```

### Tipos
| Tipo | Quando |
|---|---|
| `feat` | nova funcionalidade |
| `fix` | correção de bug |
| `refactor` | muda código sem mudar comportamento |
| `test` | adiciona/ajusta testes |
| `docs` | documentação (inclui este cofre) |
| `chore` | build, deps, config, Docker |

### Escopo = domínio
`core`, `folha`, `funcionarios`, `ponto`, `ferias`, `beneficios`, `recrutamento`, `solicitacoes`, `relatorios`, `ia`, `filament`.

## Exemplo real

```
feat(folha): calculadora de IRRF progressivo com brick/math

Implementa CalculadoraIrrf lendo faixas de TabelaIrrf por vigência.
Sem lógica no banco (D-02). Coberto por CalculadoraIrrfTest com
valores de borda de cada faixa.

Ref: [[Folha de Pagamento]] · D-07 (precisão decimal)
```

## O "caminho a seguir"

Ordem dos commits espelha o cronograma do plano (Fase 0 → 5). Cada fase começa com um commit `docs` atualizando o cofre e termina com os `feat/test` da fase. Assim o histórico conta a história do projeto na ordem em que dá pra estudar.

- Só commitar quando a [[Visão Geral|Definition of Done]] estiver satisfeita (feature não fica "meio pronta" no `main`).
- Trabalho em progresso vive em branch, não no `main`.
