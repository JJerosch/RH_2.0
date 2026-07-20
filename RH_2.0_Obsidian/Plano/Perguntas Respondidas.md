# Perguntas Respondidas (Q1–Q17)

Consolidação final das 17 perguntas do planejamento, com as respostas definitivas. Substitui o antigo `Perguntas_Laravel.txt`. Rastreamento vivo em [[Status das Perguntas]]; o "porquê" de cada uma em [[Registro de Decisões]].

> Status geral: **16 decididas · 1 adiada de propósito (Q15 — E2E).**

---

## Bloco 1 — Bloqueante para começar (Fase 0-1)

### Q1 — Perfis de acesso ✅ (D-20/21/22)
- **Perfis (lista fechada):** ADMIN, Chefe de RH, Chefe de DP, Assistente de RH, Assistente de DP, Funcionário.
- **"Gestor":** não é um 7º perfil — é **atributo** de um funcionário responsável por um departamento e/ou vaga.
- **Modelo de permissão:** perfil base (RBAC) + contexto (gestor de X), via Policies do Laravel.
- **Login:** SaaS multitenant; sem cadastro público; e-mail criado na contratação (`joão.jerosch@{tag}.com.br`); tela de login só com entrar + esqueci a senha.
- **Cadeia de contratação:** Gestor solicita vaga → Chefe de RH aprova → Chefe de RH + gestor da vaga tocam o Kanban → "prévia de funcionário" → DP efetiva. Ver [[Perfis de Acesso]].

### Q2 — 2FA (parâmetros) ✅ (D-11)
- 5 tentativas até bloquear; código expira em **10 minutos**; reenvio até 5x, depois cooldown; bloqueio com cooldown **progressivo** (1 → 2 → 4 dias). Ver [[Autenticação e 2FA]].

### Q3 — Campos do funcionário ✅ (D-17)
- Lista completa aprovada como MVP (pessoais, documentos incl. **CPF único + PIS/PASEP**, contato/endereço, vínculo, remuneração, bancários, jornada).
- **Dependentes** em tabela à parte (afetam base do IRRF). Histórico salarial via `Promocao` + activitylog. Ver [[Domínios e Módulos]].

### Q4 — Benefícios ✅
- `TipoBeneficio` com valor, recorrência e **desconto automático em folha**.
- `FuncionarioBeneficio` entra no `GeradorLancamentos` **automático todo mês**; mudança **não** altera lançamentos já feitos.

---

## Bloco 2 — Antes da Fase 1-2 (alimentam a folha)

### Q5 — Ponto ✅ (D-23/24/25)
- "Agenda" diária (quantos bateram × quantos faltam); batida liberada só em **localização válida** (multi-filial).
- Batidas **pareadas por ordem** → soma = horas; **job diário** fecha a apuração.
- **Tolerância CLT (10 min)**, configurável por tenant.
- Jornada esperada via entidade **`Escala`** ligada ao funcionário. Ver [[Domínios e Módulos]].

### Q6 — Férias ✅ (D-26/27/28)
- Fluxo: funcionário solicita → **gestor** aprova → gera solicitação ao **Chefe de RH** (ok final).
- Período aquisitivo **padrão CLT**, configurável por tenant; job de vencimento/renovação.
- Escopo MVP com **fidelidade CLT completa**: fracionamento (até 3 períodos, um ≥14 dias) + **abono pecuniário** (vender 1/3). Recibo = folha tipo `FERIAS` + 1/3.

---

## Bloco 3 — Antes da Fase 3 (recrutamento)

### Q7 — Recrutamento ✅ (D-29/30/31)
- **Kanban:** Inscrito → Triagem → Entrevista RH → Entrevista Gestor → Proposta → Contratado (+ raia **Reprovado**).
- **Prévia de funcionário** nasce na etapa **"Proposta"**; "Contratado" efetiva o `Funcionario`.
- **CPF repetido:** mostra aviso referenciando o funcionário existente (LGPD fina depois).
- **Extração por IA:** formato padrão de 9 blocos (pessoais, resumo, formação, experiência, competências, certificações, idiomas, pretensão salarial, disponibilidade). IA só sugere; RH revisa. Ver [[Inteligência Artificial]].

---

## Bloco 4 — Decide o tamanho, mas pode esperar

### Q8 — Folha (tipos) ✅ (D-16)
- Enum `tipo`: **MENSAL** / **RESCISAO** / DECIMO_TERCEIRO / FERIAS (MVP entrega os 2 primeiros).
- Reabrir folha: **não existe botão** pra quem não pode (hierarquia na UI). IA só explica folha **fechada**, nunca modifica.

### Q9 — Contratos de API ✅ (D-14)
- **Só a Folha** tem API REST (6 endpoints); resto é só Filament.
- Contratos = esqueleto fixado, **refinados campo a campo** na implementação, com feature test garantindo o formato. Ver [[Folha de Pagamento]].

### Q10 — Frontend ✅ (D-06)
- **Filament** pra todo CRUD administrativo (RH/DP); **Blade custom bonito** pro portal do funcionário (meus-holerites, ponto, férias, solicitações, perfil) + Kanban.
- Desenhar do zero, mais profissional/bonito (não copiar o Spring).

### Q11 — Relatórios ✅ (D-18)
- Formato: página **Filament** com filtros/gráficos + **CSV** em todos + **PDF** nos formais.
- Lista fechada em **6**: headcount, custo de folha, férias vencendo (MVP) + turnover, absenteísmo/ponto, aniversariantes/tempo de casa.

### Q12 — App `solicitacoes` ✅
- Fluxo de aprovação **e** log de ações importantes (uma contratação vira trilha de solicitações aprovadas). Coexiste com os fluxos específicos. Ver [[Perfis de Acesso]].

---

## Bloco 5 — Decisão de escopo

### Q13 — Auditoria / LGPD ✅ (D-13)
- **`spatie/laravel-activitylog`** (autor + diff automático nos models sensíveis). Distinto do log de negócio do Solicitacoes.

### Q14 — RNF ✅ (D-15)
- **Uso real pequeno esperado**: metas modestas de tempo de resposta, backup, índices pensados, isolamento por tenant levado a sério.

### Q15 — Testes E2E ⏸️ (adiado de propósito)
- Fica pro futuro (Dusk nativo do Laravel ou Playwright). Anotado no plano, não bloqueia o MVP.

---

## Bloco 6 — Específicos da troca de stack

### Q16 — Ambiente de PDF ✅ (D-09)
- **`spatie/laravel-pdf`** (Chromium via Browsershot), instalado por Docker (Fase 4). PDF gerado **on-demand**, nunca armazenado.

### Q17 — Confiabilidade do cliente de IA ✅ (D-08)
- Wrapper Guzzle com **camada mínima** de confiabilidade na Fase 5: timeout + retry com backoff em 429/5xx/529, máx. 3 tentativas. Ver [[IA com a Claude API]].
