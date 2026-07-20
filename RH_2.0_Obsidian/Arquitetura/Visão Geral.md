# Visão Geral

## O que é

**RHParaTodos 2.0** — sistema de RH/DP (Recursos Humanos / Departamento Pessoal) como **SaaS multitenant** (várias empresas, dados isolados por tenant). Reconstrução de portfólio de um projeto que existia incompleto em Java/Spring. Ver a decisão de stack em [[Registro de Decisões]] (D-01).

## O que o sistema faz

- **Folha de pagamento** com cálculo real de INSS/IRRF/FGTS e **rescisão** completa — o núcleo. Ver [[Folha de Pagamento]].
- **Gestão de pessoas**: funcionários, cargos, departamentos, promoções.
- **Ponto eletrônico** (batida por localização válida, multi-filial) e **férias** (por solicitação).
- **Benefícios** que descontam automaticamente na folha.
- **Recrutamento** com Kanban de candidatos.
- **Solicitações** como fluxo de aprovação + log de ações importantes.
- **Relatórios** (headcount, custo de folha, férias vencendo...).
- **2 diferenciais de IA** (Claude API): explicar holerite em linguagem natural e extrair dados de currículo. Ver [[Inteligência Artificial]].

## Stack (resumo)

| Camada | Escolha | Detalhe |
|---|---|---|
| Backend | Laravel 11 | ver [[Laravel]] |
| Banco | PostgreSQL | ver [[Banco de Dados e SQL]] |
| Painel admin (RH/DP) | Filament | ver [[Filament]] |
| Frontend (funcionário) | Blade + JS vanilla | ver [[Frontend (Blade, HTML, CSS, JS)]] |
| Auth API | Laravel Sanctum | ver [[APIs REST e Sanctum]] |
| Precisão monetária | brick/math | ver [[Precisão Decimal e Dinheiro]] |
| PDF | spatie/laravel-pdf (Chromium) | D-09 |
| IA | wrapper Guzzle → Claude API | ver [[IA com a Claude API]] |
| Testes | Pest | ver [[Testes (Pest)]] |
| Empacotamento | Docker + docker-compose | ver [[Docker]] |

## Princípio central

> **Nenhuma regra de negócio vive no banco.** Todo cálculo é service PHP testável. (D-02)

Isso nasceu do maior erro do projeto original: cálculos escondidos em triggers SQL que nunca eram chamados. Ver [[Arquitetura de Software]] pra entender o porquê arquitetural.

## Definition of Done

Uma feature só está pronta quando **simultaneamente**: migration + model, service com teste, endpoint testado, tela renderizando dados reais (não mock), e **só então** a rota é registrada. Nunca entra rota sem tela funcionando atrás.

## Mapa dos documentos

- Arquitetura detalhada por área: [[Domínios e Módulos]], [[Folha de Pagamento]], [[Autenticação e 2FA]], [[Inteligência Artificial]], [[Perfis de Acesso]].
- Por que cada escolha: [[Registro de Decisões]].
- O que estudar: [[Cronograma de Estudos]].
- O que falta decidir: [[Status das Perguntas]].
