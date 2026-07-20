# Scrum do Projeto

Como organizamos a execução do RHParaTodos 2.0. Scrum **adaptado a um projeto solo de portfólio** (você acumula os papéis; eu atuo como par de desenvolvimento/facilitador). Cada item de trabalho é uma [[Fatias Verticais|fatia vertical]].

---

## 1. Papéis (adaptação solo)

| Papel Scrum | Quem | Na prática |
|---|---|---|
| **Product Owner** | Você | Prioriza o backlog, decide escopo (foi o que fizemos nas perguntas Q1–Q17). |
| **Scrum Master** | Você + eu | Mantém o processo, remove impedimentos, cuida do cofre. |
| **Developers** | Você (eu como par) | Constrói as fatias. |

## 2. Cadência

- **Sprint de 2 semanas**, sem deadline rígido (o ritmo se adapta ao tempo de estudo — ver [[Cronograma de Estudos]]).
- Cada sprint tem **um objetivo (Sprint Goal)** claro e entrega um **incremento** que roda.
- Como é part-time/aprendizado, o que importa não é "fechar 2 semanas no relógio", e sim **fechar o Sprint Goal cumprindo a Definition of Done**.

## 3. Eventos

- **Sprint Planning** (início): escolher o Sprint Goal e puxar itens do Product Backlog pro Sprint Backlog.
- **Daily** (adaptado): um registro curto de "o que fiz / vou fazer / impedimentos" — pode ser uma linha no commit ou uma nota. Sem reunião.
- **Sprint Review** (fim): demonstrar o incremento rodando (dado real, nunca mock).
- **Sprint Retrospective** (fim): o que aprendi, o que melhorar. Vira nota se gerar decisão.

## 4. Artefatos

- **Product Backlog** → seção 6 abaixo (épicos + histórias).
- **Sprint Backlog** → os itens do sprint atual (seção 7).
- **Definition of Done** → a do [[Plano Técnico (Laravel)]] (migration+model, service+teste, endpoint+teste, tela com dado real, rota por último).
- **Incremento** → o software funcionando ao fim do sprint.

## 5. Release Plan (fases → sprints)

Espelha o cronograma do plano (~280-300h). Estimativa grosseira, ajustável.

| Release | Épicos | Fase do plano |
|---|---|---|
| **R0 — Fundação** | Setup, Auth+2FA, Multitenancy, Perfis | Fase 0 |
| **R1 — Domínio base** | Organização, Funcionários, Ponto, Férias, Benefícios | Fase 1 |
| **R2 — Núcleo de folha** | Folha (INSS/IRRF/FGTS), integração ponto/férias | Fase 2 |
| **R2.5 — Rescisão** | Rescisão completa | Fase 2.5 |
| **R3 — Recrutamento + Relatórios** | Recrutamento (Kanban), Relatórios | Fase 3 |
| **R4 — Entrega** | Docker, segurança, polimento de UI | Fase 4 |
| **R5 — IA** | Explicador de holerite, extração de currículo | Fase 5 |

## 6. Product Backlog (épicos e histórias)

Formato de história: **Como `<persona>`, quero `<ação>`, para `<valor>`.** Priorizado de cima pra baixo.

### ÉPICO 0 — Fundação (R0)
- Como **dev**, quero o projeto Laravel + Postgres + Pest rodando em Docker, para ter base testável.
- Como **plataforma**, quero isolamento por tenant (`tenant_id` + global scope), para nenhum dado vazar entre empresas. → [[Multitenancy (SaaS)]]
- Como **usuário**, quero fazer login (e "esqueci a senha"), para acessar o sistema.
- Como **usuário**, quero validar um código 2FA por e-mail, para proteger meu acesso. → [[Autenticação e 2FA]]
- Como **sistema**, quero perfis e Policies (perfil + contexto), para autorizar cada ação. → [[Perfis de Acesso]]

### ÉPICO 1 — Organização & Funcionários (R1)
- Como **DP**, quero cadastrar departamentos e cargos, para estruturar a empresa.
- Como **DP**, quero cadastrar/editar funcionários (Filament) com dependentes, para manter o cadastro. → D-17
- Como **DP**, quero registrar promoções (histórico salarial), para não perder o "de/para".

### ÉPICO 2 — Ponto (R1)
- Como **funcionário**, quero bater ponto em local válido, para registrar minha jornada. → D-23/24/25
- Como **DP**, quero ver a agenda diária (quem bateu × quem falta), para acompanhar.
- Como **sistema**, quero um job diário que apura as horas, para alimentar a folha. → [[Filas e Agendamento]]

### ÉPICO 3 — Férias (R1)
- Como **funcionário**, quero solicitar férias (inteiras/fracionadas + abono), para gozar meu direito. → D-26/27/28
- Como **gestor/Chefe RH**, quero aprovar a solicitação, para controlar o fluxo.
- Como **sistema**, quero controlar o período aquisitivo (job de vencimento), para aplicar a CLT.

### ÉPICO 4 — Benefícios (R1)
- Como **DP**, quero definir tipos de benefício e vincular a funcionários, para descontar em folha. → Q4

### ÉPICO 5 — Folha de Pagamento (R2) 🔑
- Como **DP**, quero abrir/processar/fechar a folha mensal, para pagar os funcionários. → [[Folha de Pagamento]]
- Como **sistema**, quero calcular INSS/IRRF/FGTS com `brick/math` e teste, para acertar cada centavo. → [[Precisão Decimal e Dinheiro]]
- Como **funcionário**, quero ver meus holerites (dado real) e baixar o PDF, para conferir. → D-09/D-14

### ÉPICO 6 — Rescisão (R2.5)
- Como **DP**, quero processar um desligamento com cálculo completo, para pagar as verbas rescisórias. → D-10

### ÉPICO 7 — Recrutamento (R3)
- Como **gestor**, quero solicitar abertura de vaga, para contratar. → D-22
- Como **Chefe RH/gestor da vaga**, quero mover candidatos no Kanban, para conduzir o processo. → D-29
- Como **Chefe RH**, quero revisar os dados extraídos por IA e efetivar a contratação, para admitir. → D-30/31

### ÉPICO 8 — Relatórios (R3)
- Como **RH/DP**, quero relatórios (headcount, custo de folha, férias vencendo) com filtros e export, para decidir. → D-18

### ÉPICO 9 — Solicitações (transversal)
- Como **sistema**, quero registrar toda aprovação como Solicitacao (log), para ter trilha auditável. → Q12

### ÉPICO 10 — IA (R5) 🤖
- Como **funcionário**, quero uma explicação em português do meu holerite, para entender os descontos. → [[Inteligência Artificial]]
- Como **RH**, quero que a IA extraia dados do currículo (com minha revisão), para agilizar a triagem.

### Transversais sempre ligados
- Auditoria/LGPD (`activitylog`, D-13), Segurança ([[Segurança e LGPD]]) e Testes ([[Testes (Pest)]]) acompanham **toda** fatia, não são épicos isolados.

## 7. Sprint 1 — planejamento (a fazer)

> **Sprint Goal:** *"Ambiente rodando e um usuário consegue logar com 2FA, ponta a ponta, com testes."*

Sprint Backlog (cada item = uma fatia vertical, fecha a DoD):
1. **Setup**: Laravel + Postgres **via Laravel Sail (Docker desde o início)** + Pest + CI mínimo rodando os testes. Passo a passo em [[Docker - Guia Prático]].
2. **Multitenancy base**: coluna `tenant_id` + global scope + um tenant de exemplo.
3. **Usuário + perfis**: model `Usuario`, enum de perfis, seed de um ADMIN.
4. **Login + logout** (sessão web) com tela.
5. **2FA por e-mail**: `DoisFatoresService` + middleware + Mailable + tela do código + **testes** (hash cabe no campo, expira em 10 min, bloqueia em 5 tentativas).
6. **Esqueci a senha**: fluxo de reset.

Fora do Sprint 1 (próximos): Organização/Funcionários entram no Sprint 2.

> Ao iniciar de fato, fazemos o **Sprint Planning** confirmando/ajustando esses itens e começamos pela fatia 1.
