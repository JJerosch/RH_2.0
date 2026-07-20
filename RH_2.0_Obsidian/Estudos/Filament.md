# Filament

## O que é
Um **painel administrativo** pronto pra Laravel (construído sobre Livewire + Alpine + Tailwind). Gera telas de CRUD (criar/ler/editar/excluir) com tabelas, filtros e formulários com pouca configuração. É o equivalente ao Django Admin. Decisão D-06.

## Por que neste projeto
Cobre **todo o CRUD administrativo de RH/DP** (organizacao, funcionarios, ponto, ferias, beneficios, solicitacoes, relatorios) sem você desenhar cada tela na mão. Só o portal do funcionário e o Kanban ficam em Blade custom (ver [[Frontend (Blade, HTML, CSS, JS)]]).

## Fundamentos a dominar
- **Resources** — o conceito central: uma classe que define como um model aparece no painel (tabela + formulário).
- **Forms** (schema de campos) e **Tables** (colunas, filtros, ações).
- **Relation Managers** — editar relações (ex.: dependentes de um funcionário) dentro da tela do pai.
- **Widgets** — cards e gráficos (usados nos [[Registro de Decisões|relatórios]], D-18).
- **Actions** — botões que disparam lógica (ex.: "processar folha").
- **Policies / autorização** — restringir o que cada [[Perfis de Acesso|perfil]] vê e faz.
- Noção mínima de **Livewire** (o motor por trás), só o suficiente pra não se perder.

## Onde estudar
- **Documentação oficial** (filamentphp.com/docs) — muito boa, com exemplos.
- Canal do **Filament no YouTube** e a comunidade (Discord oficial).
- Laracasts tem séries sobre Filament.

## Como saber que entendeu
Consegue criar um Resource pra `Funcionario` com formulário, tabela filtrável e um Relation Manager pra `Dependente`, respeitando as permissões por perfil.

> [!note] Custo previsto
> A curva de aprendizado do Filament está orçada na **Fase 1** do cronograma. É a peça mais "nova" pra quem vem de outro stack.
