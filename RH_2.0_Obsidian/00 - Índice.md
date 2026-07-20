# 🗂️ RHParaTodos 2.0 — Índice do Cofre

Ponto de partida de tudo. Este cofre é a **memória viva** do projeto: por que cada decisão foi tomada, como a arquitetura funciona, o que ainda falta decidir e o que estudar pra entender e corrigir cada parte.

> [!tip] Primeira vez aqui? Leia [[Como usar este cofre]].

---

## 🚀 O projeto em uma frase

Reconstrução do zero, em **Laravel**, de um sistema de RH/DP (SaaS multitenant) que existia incompleto em Java/Spring — como projeto de portfólio, com dois diferenciais de IA (Claude API). Ver [[Visão Geral]].

---

## 🧭 Navegação

### Decisões (o "porquê")
- [[Registro de Decisões]] — log de todas as decisões de arquitetura, com contexto e consequência.

### Arquitetura (o "como")
- [[Visão Geral]] — o sistema inteiro em uma página.
- [[Domínios e Módulos]] — a estrutura `app/Domain/`.
- [[Folha de Pagamento]] — o núcleo: cálculo de INSS/IRRF/FGTS, rescisão, endpoints.
- [[Autenticação e 2FA]] — login, dois fatores, parâmetros.
- [[Inteligência Artificial]] — holerite explicado + extração de currículo.
- [[Perfis de Acesso]] — quem pode o quê.

### Metodologia (o "como executar")
- [[Fatias Verticais]] — como construímos cada funcionalidade (do banco até a tela).
- [[Scrum do Projeto]] — papéis, sprints, Product Backlog e o plano do Sprint 1.

### Pendências (o "o que falta")
- [[Status das Perguntas]] — as 17 perguntas do planejamento: decididas × parkeadas.

### Estudos (o "como aprender")
- [[Cronograma de Estudos]] — roteiro de pré-requisitos, na ordem em que o projeto precisa deles.
- Destaque transversal: [[Domínio - Folha de Pagamento e CLT]] — o conhecimento de negócio que atravessa tudo.

### Plano e documentos-fonte (agora dentro do cofre, em `Plano/`)
- [[Plano Técnico (Laravel)]] — **o contrato técnico**, atualizado com as decisões Q1–Q17.
- [[Perguntas Respondidas]] — as 17 perguntas com as respostas finais.
- [[Handoff de Origem]] — histórico completo da conversa que originou o projeto.
- `Plano/_Histórico/` — plano Django e development guide antigos, mantidos só como histórico.

### Convenções
- [[Commits]] — padrão de commit que deixa um rastro estudável.

---

> [!note] Convenção de ouro
> O [[Plano Técnico (Laravel)|plano técnico]] é o **contrato** e só muda com ordem explícita. As notas de [[Registro de Decisões|Decisões]], [[Visão Geral|Arquitetura]] e [[Cronograma de Estudos|Estudos]] são a **memória viva** — atualizadas a cada decisão. Tudo agora vive dentro deste cofre (a raiz de `RH_2.0/` só tem o `.git` e o cofre).
