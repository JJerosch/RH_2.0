# Cronograma de Estudos

Roteiro dos pré-requisitos pra você **entender, defender e corrigir** cada parte do projeto — não só "fazer funcionar". A ordem segue quando o projeto precisa de cada coisa (espelha as fases do plano). Ver [[Visão Geral]].

> [!tip] Como usar
> Não precisa dominar tudo antes de começar. Estude **a camada da fase atual**, code, volte e aprofunde. Cada tópico tem sua própria nota com fundamentos e links.

> [!important] Estudo transversal (comece cedo, revisite sempre)
> **[[Domínio - Folha de Pagamento e CLT]]** — não é uma fase, é o conhecimento de negócio que atravessa o projeto inteiro. Sem ele você não sabe se o cálculo está certo nem consegue escrever os testes. Estude em paralelo desde o começo.

## Ordem recomendada

### 🟢 Base (antes de escrever qualquer linha) — Fase 0
1. **[[PHP e Composer]]** — a linguagem e o gerenciador de pacotes. Sem isso, nada.
2. **[[Git e Commits]]** — versionar e deixar rastro. Ver também [[Commits]].
3. **[[Arquitetura de Software]]** — o *porquê* das nossas regras (services, nada no banco). O que mais te diferencia numa entrevista.

### 🟡 Núcleo do framework — Fases 0–1
4. **[[Laravel]]** — o coração de tudo: rotas, Eloquent, migrations, middleware, injeção de dependência, validação, Blade.
5. **[[Banco de Dados e SQL]]** — PostgreSQL, modelagem relacional, índices. A folha depende disso.
6. **[[Multitenancy (SaaS)]]** — isolamento de dados por empresa. Fundacional: decide o desenho de quase toda tabela.
7. **[[Filament]]** — o painel admin de RH/DP. Curva de aprendizado real.

### 🟠 O núcleo do sistema — Fase 2
8. **[[Precisão Decimal e Dinheiro]]** — por que `float` é proibido e como o `brick/math` resolve.
9. **[[Testes (Pest)]]** — como provar que o cálculo de folha está certo. Inseparável do núcleo.
10. **[[APIs REST e Sanctum]]** — os endpoints de folha e autenticação por token.
11. **[[Filas e Agendamento]]** — jobs de apuração de ponto e vencimento de férias; e-mail em background.

### 🔵 Frontend e entrega — Fases 3–4
12. **[[Frontend (Blade, HTML, CSS, JS)]]** — o portal bonito do funcionário e o Kanban.
13. **[[Docker]]** — empacotar app + Postgres + Chromium; base do deploy.
14. **[[Segurança e LGPD]]** — 2FA, OWASP básico, auditoria, dados pessoais.

### 🟣 O diferencial — Fase 5
15. **[[IA com a Claude API]]** — HTTP client, prompt, tool use / JSON estruturado, prompt injection.

## Regra de estudo eficiente
Para cada tópico, pare quando conseguir responder: **"consigo explicar o que é, por que está no projeto, e debugar um erro simples nele?"** Isso é o suficiente pra avançar — o resto vem com o uso.

## Filosofia
O objetivo não é decorar sintaxe, é **entender os conceitos** por trás. Framework muda; fundamento (HTTP, SQL, OOP, arquitetura) fica. É o fundamento que te deixa dizer com verdade "eu fiz e entendo este projeto".
