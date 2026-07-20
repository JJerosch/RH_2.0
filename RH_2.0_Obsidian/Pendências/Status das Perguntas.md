# Status das Perguntas

As 17 perguntas do planejamento. Fonte original: `Perguntas_Laravel.txt` na raiz de `RH_2.0/`. Decisões refletidas em [[Registro de Decisões]].

## ✅ Decididas

| # | Tema | Decisão | ADR |
|---|---|---|---|
| Q1 | Perfis / permissão | Perfil base + contexto; "Gestor" é atributo; cadeia de contratação confirmada | D-20/21/22 · [[Perfis de Acesso]] |
| Q2 | 2FA — parâmetros | 5 tentativas, 10 min, reenvio 5x, cooldown 1→2→4 dias | D-11 · [[Autenticação e 2FA]] |
| Q3 | Campos do funcionário | Lista completa como MVP; dependentes em tabela à parte | D-17 · [[Domínios e Módulos]] |
| Q4 | Benefícios | Descontam automático na folha; mudança não altera lançamentos já feitos | — |
| Q5 | Ponto | Pareamento por ordem + job diário; tolerância CLT 10 min configurável; jornada via Escala | D-23/24/25 · [[Domínios e Módulos]] |
| Q6 | Férias | Fluxo Func→Gestor→Chefe RH; aquisitivo CLT configurável; MVP com fracionamento + abono | D-26/27/28 · [[Domínios e Módulos]] |
| Q7 | Recrutamento | Kanban de 6 etapas + Reprovado; prévia na "Proposta"; formato padrão de extração por IA | D-29/30/31 · [[Domínios e Módulos]] |
| Q8 | Tipo de folha | enum MENSAL/RESCISAO/DECIMO_TERCEIRO/FERIAS (MVP: 2) | D-16 |
| Q9 | API REST | Só folha tem API; contratos = esqueleto + refino na implementação | D-14 · [[Folha de Pagamento]] |
| Q10 | Telas | Filament (RH/DP) + portal Blade (funcionário/candidato) | D-06 |
| Q11 | Relatórios | Filament + CSV + PDF nos formais; lista fechada em 6 (3 no MVP) | D-18 |
| Q12 | Solicitações | Fluxo de aprovação **e** log de ações importantes | [[Perfis de Acesso]] |
| Q13 | Auditoria/LGPD | spatie/laravel-activitylog | D-13 |
| Q14 | RNF | Uso real pequeno esperado (metas modestas, backup, índices) | D-15 |
| Q16 | PDF | spatie/laravel-pdf (Chromium via Docker) | D-09 |
| Q17 | IA retry | Camada mínima (timeout + backoff) na Fase 5 | D-08 |

## ⏸️ Parkeadas (decisão consciente de adiar)

| # | Tema | Situação |
|---|---|---|
| Q15 | Testes E2E | **Adiado de propósito** pro futuro (Dusk ou Playwright), já anotado no plano. Não é bloqueante. |

## 🎯 Situação
**Todas as pendências de design fechadas.** Só o Q15 (E2E) fica adiado conscientemente — não bloqueia começar o projeto.

Detalhes que viram trabalho na hora da implementação (não são pendências abertas, só refinamento com teste garantindo):
- Contratos exatos dos endpoints de folha (D-14) → definidos campo a campo na Fase 2.
- LGPD fina do aviso de CPF repetido no recrutamento → refinar quando implementar o Recrutamento.

## Próximo passo
Persistir as decisões nos documentos-fonte (`Perguntas_Laravel.txt` e `RHParaTodos-Laravel-plano.md`) e/ou começar a Fase 0. Ver [[Cronograma de Estudos]] pra o que estudar primeiro.
