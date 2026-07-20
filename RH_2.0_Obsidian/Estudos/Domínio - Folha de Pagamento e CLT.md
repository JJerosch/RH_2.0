# Domínio — Folha de Pagamento e CLT

## O que é
O **conhecimento de negócio** por trás do sistema: como funciona uma folha de pagamento no Brasil e o que a **CLT** (Consolidação das Leis do Trabalho) exige. Não é tecnologia — é a regra que o código precisa acertar.

## Por que neste projeto
É provavelmente o pré-requisito **mais importante e mais esquecido**. Você pode dominar Laravel e ainda assim calcular a folha errado se não entender INSS, IRRF, rescisão e férias. Sem esse domínio, você não consegue nem escrever os testes ([[Testes (Pest)]]) — porque não sabe qual é o resultado certo. É o coração da [[Folha de Pagamento]].

## Fundamentos a dominar
- **Proventos × descontos × informativos**: o que soma, o que subtrai, o que só aparece (ex.: FGTS é informativo no holerite).
- **INSS** (contribuição previdenciária): tabela **progressiva por faixas**, com teto. Muda todo ano.
- **IRRF** (imposto de renda retido): tabela progressiva, **dedução por dependente**, base = salário − INSS − deduções.
- **FGTS**: 8% sobre a remuneração, depositado pelo empregador (não desconta do funcionário).
- **13º salário**: proporcional aos meses trabalhados.
- **Férias**: período aquisitivo (12 meses → 30 dias), **1/3 constitucional**, fracionamento, abono pecuniário (vender 1/3). Ver decisões D-26/27/28.
- **Rescisão**: saldo de salário, férias vencidas/proporcionais + 1/3, 13º proporcional, aviso prévio (indenizado/trabalhado), **multa do FGTS** (40% sem justa causa, 20% acordo). O que cada **tipo de desligamento** gera (ver `RegrasPorTipoDesligamento`).
- **Jornada e horas**: jornada normal, hora extra, adicional, tolerância (10 min/dia). Liga com o módulo Ponto.

## Onde estudar
- **Portal do eSocial** e **gov.br/trabalho** — fontes oficiais das regras.
- Tabelas de **INSS e IRRF do ano vigente** (Receita Federal / Previdência) — você vai precisar dos números reais.
- Livros/cursos de **Departamento Pessoal** e **rotinas trabalhistas** (há bons cursos introdutórios online).
- Blogs de contabilidade explicam cálculo de rescisão e férias com exemplos numéricos — ótimos pra montar casos de teste.

## Como saber que entendeu
Consegue calcular à mão o holerite de um salário exemplo (INSS → base IRRF → IRRF → líquido) e uma rescisão sem justa causa completa — e usar esses números como **casos de teste** das calculadoras. Ver [[Folha de Pagamento]].

> [!warning] Cuidado com o "ano vigente"
> As tabelas de INSS/IRRF mudam por lei quase todo ano. Por isso elas vivem em tabela de banco com `vigente_desde` (não em código fixo) — é exatamente a razão da decisão D-02 ([[Arquitetura de Software]]).
