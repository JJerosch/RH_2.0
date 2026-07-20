# Folha de Pagamento

O **núcleo** da reconstrução. É aqui que o projeto original mais falhava (cálculo em triggers SQL nunca chamadas) e onde a regra D-02 ("nada de negócio no banco") mais importa. Ver [[Registro de Decisões]].

## Models (só dados)

- **FolhaPagamento** — `competencia`, `tipo` (enum D-16: MENSAL/RESCISAO/DECIMO_TERCEIRO/FERIAS), `status` (ABERTA/FECHADA), totais, `fechada_em/por`.
- **LancamentoFolha** — cada linha do holerite: `tipo` (PROVENTO/DESCONTO/INFORMATIVO), `codigo`, `descricao`, `referencia`, `valor`, `origem`.
- Tabelas de parâmetros: `TabelaInss`, `TabelaIrrf`, `ParametroIrrf` — com `vigente_desde`, faixas, alíquota, parcela a deduzir. **Sem nenhuma function SQL.**

## Services (toda a lógica, 100% testável)

```
Folha/Services/
├── CalculadoraInss.php   # calcular(BigDecimal base, int ano): BigDecimal
├── CalculadoraIrrf.php   # calcular(BigDecimal baseLiquida, Carbon data): BigDecimal
├── CalculadoraFgts.php
├── GeradorLancamentos.php  # monta os LancamentoFolha (salário, benefícios, ponto, férias)
└── ProcessadorFolha.php    # orquestra: abrir → gerar → calcular → totalizar → fechar
```

Todo cálculo usa **`brick/math`** (nunca `float` — ver [[Precisão Decimal e Dinheiro]]).

### Integração explícita com Ponto e Férias
No original, Ponto e Férias eram desconectados da folha. Aqui a integração tem dono:
- `GeradorLancamentos::gerarLancamentosPonto()` — lê `PontoApuracaoDiaria`, cria HEXTRA/FALTA.
- `GeradorLancamentos::gerarLancamentosFerias()` — cria FERIAS/FERIAS_1_3 quando uma `FeriasRegistro` é aprovada.
- Ambos chamados **explicitamente** por `ProcessadorFolha` (nunca implícito).

### Fechamento/reabertura
`ProcessadorFolha` roda em transação com `lockForUpdate`. Reprocessar folha FECHADA lança `FolhaJaFechadaException`. Reabrir só ADMIN/DP_CHEFE.

## Rescisão (D-10)

`Folha/Services/Rescisao/` — cálculo financeiro completo:
```
├── CalculadoraSaldoSalario.php
├── CalculadoraFeriasRescisao.php        # vencidas + proporcionais, ambas com 1/3
├── CalculadoraDecimoTerceiroProporcional.php
├── CalculadoraAvisoPrevio.php           # indenizado / trabalhado / dispensado
├── CalculadoraMultaFgts.php             # 40% sem justa causa, 20% acordo, 0% resto
├── RegrasPorTipoDesligamento.php        # tabela explícita do que cada tipo gera
└── ProcessadorRescisao.php              # gera FolhaPagamento tipo RESCISAO, fecha automático
```

`RegrasPorTipoDesligamento::REGRAS` é uma **estrutura de dados** (array), não `if/else` espalhado — cada tipo (SEM_JUSTA_CAUSA, PEDIDO_DEMISSAO, JUSTA_CAUSA, ACORDO) diz quais verbas se aplicam. Reaproveita `CalculadoraInss/Irrf` da folha mensal.

## API (D-14 — só a Folha tem API REST)

| Endpoint | O quê |
|---|---|
| `GET /api/folha/meus-holerites` | lista resumida do funcionário logado |
| `GET /api/folha/{id}` | detalhe + lançamentos |
| `POST /api/folha/{id}/processar` | roda ProcessadorFolha (RH/DP) · 409 se fechada |
| `POST /api/folha/{id}/fechar` | fecha (RH/DP) |
| `POST /api/folha/{id}/reabrir` | ADMIN/DP_CHEFE · 403 por hierarquia |
| `GET /api/folha/{id}/pdf` | PDF on-demand (D-09) |
| `POST /api/folha/{id}/explicar` | explicação por IA, só se FECHADA → [[Inteligência Artificial]] |

Contratos = esqueleto fixado, campos refinados na implementação com feature test garantindo o formato.

## Testes obrigatórios (o que o original nunca teve)
- `CalculadoraIrrfTest` / `CalculadoraInssTest`: cada faixa, valores de borda, base zero/negativa.
- `ProcessadorFolhaTest`: abrir→processar→fechar; reprocessar fechada lança exceção.
- `RegrasPorTipoDesligamentoTest` e `ProcessadorRescisaoTest`.
- `ApiFolhaTest`: endpoint retorna os mesmos valores dos services (sem lógica duplicada no controller).
