# 📌 Progresso do Projeto — RHParaTodos 2.0

> [!abstract] Leia isto primeiro
> Esta é a **foto do "onde estamos agora"**. Se você (ou uma nova conversa) quer saber o estado do projeto sem ler as 30+ notas, comece aqui. O [[00 - Índice]] diz *o que existe no cofre*; **este arquivo diz o que já foi feito e qual é o próximo passo.**

**Última atualização:** 2026-07-21
**Fase atual:** Planejamento concluído → **prestes a iniciar o Sprint 1 (código)**
**Nenhuma linha de código escrita ainda** — o projeto Laravel ainda não existe.

---

## 🎯 Onde estamos, em uma frase

Todo o **planejamento e a base de conhecimento estão prontos** (17 perguntas decididas, 32 ADRs, arquitetura documentada, metodologia e Scrum definidos, cronograma de estudos escrito). O próximo passo concreto é a **fatia 1 do Sprint 1**: criar o projeto Laravel + Postgres via **Laravel Sail** (Docker desde o início).

---

## ✅ FASE 0 — Planejamento e Fundação do Conhecimento (CONCLUÍDA)

### Decisões de produto/escopo
- [x] 17 perguntas de planejamento fechadas (16 decididas, Q15/E2E adiada conscientemente) → [[Status das Perguntas]]
- [x] 32 decisões de arquitetura registradas (D-01 a D-32) → [[Registro de Decisões]]
- [x] Escopo do MVP definido (campos, módulos, fluxos) → [[Perguntas Respondidas]]

### Documentação de arquitetura (o "como")
- [x] [[Visão Geral]] — o sistema inteiro em uma página
- [x] [[Domínios e Módulos]] — estrutura `app/Domain/`
- [x] [[Folha de Pagamento]] — núcleo de cálculo (INSS/IRRF/FGTS), rescisão, endpoints
- [x] [[Autenticação e 2FA]]
- [x] [[Inteligência Artificial]] — holerite explicado + extração de currículo
- [x] [[Perfis de Acesso]] — RBAC + contexto via Policies

### Metodologia e processo (o "como executar")
- [x] [[Fatias Verticais]] — método de desenvolvimento registrado
- [x] [[Scrum do Projeto]] — papéis, cadência, Product Backlog (10 épicos), Release Plan
- [x] Definition of Done definida (migration+model → service+teste → endpoint+teste → tela com dado real → rota por último)

### Base de estudos (o "como aprender")
- [x] [[Cronograma de Estudos]] — roteiro de pré-requisitos na ordem em que o projeto precisa deles
- [x] 17 notas de estudo escritas (PHP, Git, Arquitetura, Laravel, SQL, Filament, Precisão Decimal, Pest, APIs/Sanctum, Frontend, Docker, Docker Prático, Segurança/LGPD, IA, CLT/Folha, Multitenancy, Filas)

### Organização do cofre
- [x] Cofre Obsidian criado como memória viva do projeto → [[Como usar este cofre]]
- [x] Documentos-fonte externos migrados pra dentro do cofre (`Plano/`)
- [x] [[Plano Técnico (Laravel)]] atualizado com as decisões Q1–Q17 (o "contrato")
- [x] Raiz do repositório limpa (só `.git` + o cofre)
- [x] Commits feitos e enviados pro GitHub (`5f8a062`, `a15a788`)

---

## 🔜 SPRINT 1 — Fundação em código (A FAZER — próximo passo)

> **Sprint Goal:** *"Ambiente rodando e um usuário loga com 2FA, ponta a ponta, com testes."*
> Cada item é uma [[Fatias Verticais|fatia vertical]] e só fecha cumprindo a Definition of Done.

- [ ] **Fatia 1 — Setup** ⭐ *(começa aqui)*
  - [ ] Docker Desktop instalado e rodando (Níveis 0–2 do [[Docker - Guia Prático]])
  - [ ] Projeto Laravel criado via Sail com Postgres (`laravel.build/rhparatodos?with=pgsql`)
  - [ ] `sail up -d` sobe app + Postgres
  - [ ] Pest instalado e um teste "hello" passando
  - [ ] CI mínimo rodando os testes
- [ ] **Fatia 2 — Multitenancy base**: coluna `tenant_id` + global scope + 1 tenant de exemplo → [[Multitenancy (SaaS)]]
- [ ] **Fatia 3 — Usuário + perfis**: model `Usuario`, enum de perfis, seed de um ADMIN → [[Perfis de Acesso]]
- [ ] **Fatia 4 — Login + logout** (sessão web) com tela real
- [ ] **Fatia 5 — 2FA por e-mail**: `DoisFatoresService` + middleware + Mailable + tela do código + testes (hash cabe no campo, expira em 10 min, bloqueia em 5 tentativas) → [[Autenticação e 2FA]]
- [ ] **Fatia 6 — Esqueci a senha**: fluxo de reset

---

## 🗺️ ROADMAP — Releases seguintes (visão macro)

Detalhe completo (histórias por épico) em [[Scrum do Projeto]]. Aqui só o mapa de alto nível.

- [ ] **R0 — Fundação**: Setup, Auth+2FA, Multitenancy, Perfis *(← Sprint 1 está aqui)*
- [ ] **R1 — Domínio base**: Organização, Funcionários, Ponto, Férias, Benefícios
- [ ] **R2 — Núcleo de folha**: Folha (INSS/IRRF/FGTS), integração ponto/férias 🔑
- [ ] **R2.5 — Rescisão**: cálculo completo de verbas rescisórias
- [ ] **R3 — Recrutamento + Relatórios**: Kanban de vagas + relatórios com filtros/export
- [ ] **R4 — Entrega**: Dockerfile próprio + Chromium (PDF), segurança, polimento de UI
- [ ] **R5 — IA**: explicador de holerite + extração de currículo 🤖

---

## 🧭 Pendências conscientes

- [ ] **Q15 — Testes E2E**: adiada de propósito (decidir a ferramenta mais perto da entrega) → [[Status das Perguntas]]
- [ ] **Segredos do projeto Java antigo** (senha de app do Gmail, JWT secret) ainda precisam ser **rotacionados** antes de qualquer reuso → [[Segurança e LGPD]]

---

## 📇 Como retomar numa nova conversa

Se abrir uma conversa nova, aponte a IA para:
1. **Este arquivo** ([[Progresso do Projeto]]) — o estado atual e o próximo passo.
2. [[00 - Índice]] — o mapa de tudo.
3. [[Plano Técnico (Laravel)]] — o contrato técnico (só muda com ordem explícita).
4. [[Registro de Decisões]] — o porquê de cada escolha.

**Convenções que a IA deve respeitar:** responder em português (Brasil); nunca editar o plano técnico sem ordem explícita; commit/push só quando você pedir.
