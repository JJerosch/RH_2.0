# Arquitetura de Software

## O que é
Os princípios de **como organizar o código** pra ele ser testável, sustentável e correto. Não é uma tecnologia — é a forma de pensar que justifica as maiores decisões do projeto.

## Por que neste projeto
É o que **mais te diferencia**. Qualquer um instala Laravel; poucos sabem dizer *por que* a regra de negócio não vai no banco (D-02) ou por que o cálculo fica num service e não no controller. Ver [[Registro de Decisões]].

## Fundamentos a dominar
- **Separação de responsabilidades (SoC)**: model = dados; service = regra de negócio; controller = orquestra HTTP; view = apresentação. Por que misturar isso vira dívida técnica.
- **Por que lógica de negócio não vai no banco**: o erro nº 1 do projeto original foram cálculos em triggers SQL nunca chamadas. Regra que muda por lei (INSS/IRRF) precisa estar em código revisável e testável. É a essência da [[Folha de Pagamento]].
- **Injeção de dependência (DI)** e o **service container** do Laravel: por que `ProcessadorFolha` recebe as calculadoras no construtor em vez de criá-las.
- **Domain-oriented structure**: por que `app/Domain/{Modulo}/` (ver [[Domínios e Módulos]]).
- **Testabilidade como critério de design**: código fácil de testar costuma ser bem desenhado. Liga com [[Testes (Pest)]].
- Noções de **SOLID** (sobretudo Single Responsibility e Dependency Inversion) — sem virar dogma.

## Onde estudar
- **"Clean Architecture"** e **"Clean Code"** (Robert C. Martin) — conceitos atemporais.
- Laravel docs → **"Service Container"** e **"Service Providers"** (aplica DI na prática).
- **refactoring.guru** — design patterns e code smells, com exemplos visuais.
- Canal/artigos sobre **Domain-Driven Design (DDD)** em nível introdutório (não precisa do livro inteiro do Evans agora).

## Como saber que entendeu
Consegue explicar, sem olhar, **por que** o cálculo de folha é um service testável e não uma trigger — e quais problemas do projeto original isso resolve.
