# Registro de Decisões (ADR)

Log de todas as decisões de arquitetura. Formato de cada uma: **Contexto → Decisão → Consequência → Fonte**. Ordenado por tema, não por data. Quando decidir algo novo, adicione aqui **antes** de codar.

> Legenda de origem: `[Plano]` = veio do `RHParaTodos-Laravel-plano.md`. `[Qn]` = resposta da pergunta n em [[Status das Perguntas]].

---

## Fundamentais

### D-01 — Stack: Laravel (pivô de Django) `[Plano]`
- **Contexto:** o projeto seria em Django; antes disso existia uma versão incompleta em Java/Spring. Decisão de portfólio: testar uma stack nova, ganhando produtividade.
- **Decisão:** reconstruir tudo em **Laravel 11 + Blade + JS vanilla**, sem React. PostgreSQL mantido.
- **Consequência:** o plano Django virou histórico (não apagado). A lógica de negócio é idêntica; só muda a implementação. Ver [[Visão Geral]].

### D-02 — Nenhuma regra de negócio no banco `[Plano]`
- **Contexto:** no projeto Java, ~15 functions/triggers PL/pgSQL calculavam a folha e **nunca eram chamadas** pela aplicação — código morto. Tabela de INSS/IRRF muda todo ano por lei.
- **Decisão:** o banco só guarda schema + constraints + índices. Todo cálculo vive em **services PHP puros**, testáveis. Nada de trigger/function.
- **Consequência:** regras revisáveis por PR e cobertas por teste unitário. É a espinha dorsal da [[Folha de Pagamento]].

### D-03 — PostgreSQL (não MySQL) `[Plano]`
- **Contexto:** Laravel usa MySQL por padrão; o projeto original usava Postgres.
- **Decisão:** manter **PostgreSQL** pra facilitar comparar as três versões (Spring / Django-planejado / Laravel).
- **Consequência:** usar tipos e recursos do Postgres (ex.: coluna `json` nativa na extração de currículo).

### D-04 — Sem React `[Plano]`
- **Contexto:** decisão carregada desde a primeira versão do plano.
- **Decisão:** frontend em **Blade + JS vanilla (`fetch`)**. Sem SPA.
- **Consequência:** menos complexidade de build; telas server-side. Ver [[Domínios e Módulos]].

---

## Estrutura e ferramentas

### D-05 — Módulos manuais `app/Domain/` (não `nwidart/laravel-modules`) `[Plano]`
- **Contexto:** Laravel não tem "apps" isolados como o Django. O pacote `nwidart` foi considerado.
- **Decisão:** replicar a separação por domínio com **pastas manuais** `app/Domain/{Modulo}/`, sem dependência de terceiro.
- **Consequência:** simples pra um dev só; sem service providers extras. Ver [[Domínios e Módulos]].

### D-06 — Painel admin: Filament; funcionário/candidato: Blade custom `[Q10]`
- **Contexto:** Laravel não tem admin nativo (o Django tinha). RH/DP precisa de CRUD produtivo; o funcionário comum precisa de UI bonita e profissional.
- **Decisão:** **Filament** cobre todo CRUD administrativo (organizacao, funcionarios, ponto, ferias, beneficios, solicitacoes, relatorios). **Blade custom** cobre o portal do funcionário (meus-holerites, bater ponto, solicitar férias, minhas solicitações, meu perfil) e o Kanban de recrutamento.
- **Consequência:** curva de aprendizado do Filament entra na Fase 1. Mais telas Blade que o plano original previa.

### D-07 — Precisão decimal: `brick/math` (BigDecimal) `[Plano]`
- **Contexto:** PHP não tem `Decimal` nativo. `float` em folha causa erro de arredondamento — inaceitável.
- **Decisão:** usar **`brick/math`** (objetos imutáveis de precisão arbitrária) em todo cálculo monetário.
- **Consequência:** disciplina de nunca usar `float` pra dinheiro. Ver [[Precisão Decimal e Dinheiro]].

### D-08 — Cliente de IA: wrapper Guzzle + retry mínimo `[Plano]` `[Q17]`
- **Contexto:** **não existe SDK oficial da Anthropic em PHP** (só Python/TS/Java/Go/Ruby). A API pode responder 429 (rate limit), 529 (overloaded), 5xx ou dar timeout.
- **Decisão:** wrapper fino sobre **Guzzle** contra a REST API. Na Fase 5, incluir **camada mínima**: timeout explícito + retry com backoff exponencial em 429/5xx/529, máx. 3 tentativas. Nada de circuit breaker/fila (exagero pro volume).
- **Consequência:** replica o essencial do que o SDK Python faz sozinho. Ver [[Inteligência Artificial]].

### D-09 — PDF: `spatie/laravel-pdf` (Chromium), gerado dinamicamente `[Plano]` `[Q16]`
- **Contexto:** holerite precisa ficar bonito (CSS moderno). `dompdf` tem CSS limitado (sem flexbox/grid). `spatie/laravel-pdf` usa Chromium headless via Browsershot.
- **Decisão:** **`spatie/laravel-pdf`**, com Chromium instalado via Docker (Fase 4). O PDF é **renderizado on-demand a cada download**, nunca armazenado — a fonte (`LancamentoFolha` de folha fechada) é imutável.
- **Consequência:** ambiente precisa de Node+Chromium. Evita storage. Ressalva: mudar o template muda a aparência de holerites antigos reimpressos (valores continuam corretos).

---

## Regras de negócio

### D-10 — Rescisão com cálculo financeiro completo `[Plano]`
- **Contexto:** no projeto Java, desligar só mudava o status pra INATIVO — zero cálculo.
- **Decisão:** rescisão é um processamento de folha de verdade: saldo de salário, férias (vencidas + proporcionais + 1/3), 13º proporcional, aviso prévio, multa FGTS — variando por tipo de desligamento.
- **Consequência:** `RegrasPorTipoDesligamento` como estrutura de dados explícita e testável. Ver [[Folha de Pagamento]].

### D-11 — 2FA recriado, corrigindo o bug de schema `[Plano]` `[Q2]`
- **Contexto:** no Java o 2FA estava bem desenhado, mas gravava um hash bcrypt (~60 chars) numa coluna `varchar(10)` — quebrava sempre.
- **Decisão:** mesma arquitetura (gerar código → e-mail → bloquear rota até validar), coluna `codigo_hash string(255)`. Parâmetros: **5 tentativas, expira em 10 min, reenvio até 5x, cooldown progressivo (1→2→4 dias)**.
- **Consequência:** regra: nunca dimensionar campo de hash pelo tamanho do dado original. Ver [[Autenticação e 2FA]].

### D-12 — IA nunca decide valor; revisão humana obrigatória `[Plano]`
- **Contexto:** diferencial de IA não pode virar risco. IA que "decide" salário ou cria cadastro sozinha é inaceitável.
- **Decisão:** IA **só explica** números que o service determinístico já produziu (holerite fechado), e **só sugere** dados de currículo — RH confirma antes de qualquer `Candidato`/`Candidatura` ser salvo.
- **Consequência:** toda feature de IA tem uma etapa de revisão humana. Ver [[Inteligência Artificial]].

### D-13 — Auditoria de dados: `spatie/laravel-activitylog` `[Q13]`
- **Contexto:** LGPD pede rastrear quem alterou dado pessoal (salário, CPF). Isso é diferente do log de negócio do módulo Solicitacoes.
- **Decisão:** usar **`spatie/laravel-activitylog`** (trait nos models sensíveis grava autor + diff automaticamente). Solicitacoes = log de fluxo; activitylog = auditoria técnica.
- **Consequência:** justificado agora que há expectativa de uso real (ver D-15).

---

## Escopo e API

### D-14 — API REST só na Folha; resto só Filament `[Q9]`
- **Contexto:** os módulos admin já têm UI no Filament. Duplicar em API JSON seria retrabalho sem uso concreto.
- **Decisão:** só a **Folha** expõe API REST (6 endpoints), porque alimenta telas Blade do funcionário. Contratos = esqueleto fixado, refinados campo a campo na implementação (não travados no papel).
- **Consequência:** menos superfície de teste. Ver [[Folha de Pagamento]].

### D-15 — RNF: uso real pequeno esperado `[Q14]`
- **Contexto:** o sistema é SaaS multitenant; a ambição não é só demo local.
- **Decisão:** planejar pra **algumas empresas reais** usarem: metas modestas de tempo de resposta, backup, índices pensados, isolamento por tenant levado a sério. Sem SLA agressivo.
- **Consequência:** multitenancy é levada a sério no design desde o início.

### D-16 — Tipos de `FolhaPagamento` `[Q8]`
- **Decisão:** enum `MENSAL` / `RESCISAO` / `DECIMO_TERCEIRO` / `FERIAS`. MVP entrega só os **dois primeiros**; os outros ficam como marcador de roadmap.

### D-17 — Campos do Funcionário `[Q3]`
- **Decisão:** lista completa aprovada como MVP (pessoais, documentos incl. PIS/PASEP, contato/endereço, vínculo, remuneração, bancários, jornada). **Dependentes** em tabela à parte (afetam base do IRRF). Histórico salarial via `Promocao` + activitylog. Detalhe em [[Domínios e Módulos]].

### D-18 — Relatórios: Filament + CSV + PDF nos formais `[Q11]`
- **Decisão:** base = página Filament com filtros/gráficos; export CSV em todos; PDF só nos formais. Lista fechada em **6**: headcount, custo de folha, férias vencendo (MVP) + turnover, absenteísmo/ponto, aniversariantes/tempo de casa (depois).

### D-19 — Prazo e estimativa `[Plano]`
- **Decisão:** 7-8 meses, **sem prazo rígido**. Estimativa ~280-300h. README comparativo, seed de demo e deploy ficam fora do plano por ora.

---

## Perfis e permissões

### D-20 — Modelo de permissão: perfil base + contexto `[Q1]`
- **Contexto:** só role fixa não modela "gestor aprova apenas o próprio time".
- **Decisão:** permissão = **RBAC (perfil)** + **vínculos contextuais** (gestor de departamento/vaga), combinados via **Policies do Laravel**.
- **Consequência:** cada ação sensível tem Policy que checa role E contexto. Ver [[Perfis de Acesso]].

### D-21 — "Gestor" é atributo, não perfil `[Q1]`
- **Contexto:** gestor manda solicitações e toca o Kanban da vaga dele, mas também é funcionário comum.
- **Decisão:** **não** criar 7º perfil. "Gestor" = `Funcionario` marcado como responsável por um departamento e/ou vaga.
- **Consequência:** evita acúmulo de papéis; poder extra só no contexto do que lidera.

### D-22 — Cadeia de aprovação de contratação `[Q1c]`
- **Decisão:** Gestor solicita vaga → Chefe de RH aprova → Chefe de RH + gestor da vaga tocam o Kanban → candidato aprovado vira "prévia de funcionário" → DP efetiva o `Funcionario`. Cada passo = uma `Solicitacao` (log).
- **Consequência:** trilha auditável da contratação. Detalhes do Kanban seguem no Q7 ([[Status das Perguntas]]).

---

## Ponto

### D-23 — Apuração: pareamento por ordem + job diário `[Q5a]`
- **Contexto:** num dia há várias batidas (entrada, almoço, saída). Precisa virar horas trabalhadas.
- **Decisão:** batidas **ordenadas por horário e pareadas** (entrada↔saída); soma dos intervalos = horas do dia. Um **job diário** fecha a `PontoApuracaoDiaria` do dia anterior.
- **Consequência:** precisa de scheduler (Laravel Task Scheduling). Ver [[Domínios e Módulos]].

### D-24 — Tolerância: padrão CLT (10 min), configurável por tenant `[Q5b]`
- **Decisão:** 10 min/dia como padrão (regra CLT), **configurável por empresa**. Dentro da tolerância não gera atraso nem extra.
- **Consequência:** casa com o SaaS multitenant (D-15) — parâmetro por tenant.

### D-25 — Jornada via entidade `Escala` ligada ao funcionário `[Q5c]`
- **Contexto:** só "carga semanal" não modela turnos/intervalos.
- **Decisão:** entidade **`Escala`** (dias + horários) atribuída ao **funcionário**. A apuração compara trabalhado × escala.
- **Consequência:** suporta turnos diferentes e multi-filial. Atraso/falta/extra viram `LancamentoFolha` via `GeradorLancamentos` ([[Folha de Pagamento]]).

---

## Férias

### D-26 — Fluxo: Funcionário → Gestor → Chefe de RH `[Q6a]`
- **Decisão:** funcionário solicita pelo portal; **gestor** aprova (contexto D-20); ao aprovar, gera `Solicitacao` ao **Chefe de RH** pro ok final. Duas aprovações.
- **Consequência:** liga com [[Perfis de Acesso]] e o módulo Solicitacoes.

### D-27 — Período aquisitivo: padrão CLT, configurável por tenant `[Q6b]`
- **Decisão:** 12 meses = 30 dias; concessivo de 12 meses; "férias em dobro" se vencer; **job** monitora vencimento e abre novo período. Parâmetros por empresa.
- **Consequência:** `FeriasPeriodoAquisitivo` guarda saldo; casa com multitenant (D-15).

### D-28 — Escopo: CLT completa (fracionamento + abono) `[Q6c]`
- **Contexto:** o usuário optou pela fidelidade máxima à CLT.
- **Decisão:** **fracionamento** em até 3 períodos (um ≥14 dias) + **abono pecuniário** (vender 1/3). Recibo = folha tipo `FERIAS` com 1/3 constitucional; abono como lançamento próprio.
- **Consequência:** mais regras de validação e cálculo na folha — custo de horas maior que o MVP mínimo, aceito de propósito. Ver [[Folha de Pagamento]].

---

## Recrutamento

### D-29 — Etapas do Kanban `[Q7a]`
- **Decisão:** `Inscrito → Triagem → Entrevista RH → Entrevista Gestor → Proposta → Contratado`, com raia **Reprovado** acessível de qualquer etapa.
- **Consequência:** só Chefe de RH + gestor da vaga movem cartões (D-20). Ver [[Domínios e Módulos]].

### D-30 — Gatilho da "prévia de funcionário" `[Q7b]`
- **Decisão:** ao entrar em **"Proposta"**, cria-se a prévia; DP completa os dados; mover pra **"Contratado"** efetiva o `Funcionario`.
- **Consequência:** dá janela pro DP agir antes de efetivar. Liga com a cadeia de contratação (D-22).

### D-31 — Formato padrão da extração por IA `[Q7c]`
- **Decisão:** blocos fixos pra toda vaga — pessoais, resumo/objetivo, formação, experiência, competências, certificações, idiomas, pretensão salarial, disponibilidade/mobilidade. Campos ausentes ficam vazios (nunca inventados).
- **Consequência:** IA só sugere; RH revisa antes de persistir (D-12). Detalhe em [[Inteligência Artificial]].
