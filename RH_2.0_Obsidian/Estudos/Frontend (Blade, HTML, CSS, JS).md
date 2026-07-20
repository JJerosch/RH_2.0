# Frontend (Blade, HTML, CSS, JS)

## O que é
A camada visual. **HTML** estrutura, **CSS** estiliza, **JavaScript** dá interatividade. **Blade** é o motor de templates do Laravel que gera o HTML no servidor. Sem React (decisão D-04).

## Por que neste projeto
O **portal do funcionário** (meus-holerites, bater ponto, solicitar férias, minhas solicitações, meu perfil) e o **Kanban de recrutamento** são Blade custom — e você quer que fiquem **profissionais e bonitos** (D-06). O Filament cuida do resto.

## Fundamentos a dominar
- **HTML semântico** — estrutura correta (`<form>`, `<table>`, `<nav>`...).
- **CSS moderno** — **Flexbox e Grid** (layout), variáveis CSS, responsividade. É o que o `spatie/laravel-pdf` também renderiza no holerite (D-09).
  - Vale escolher um utilitário: **Tailwind CSS** (que o Filament já usa) mantém consistência visual.
- **Blade** — `@if`, `@foreach`, `@extends`/`@section` (layouts), componentes Blade, como passar dados do controller pra view.
- **JavaScript vanilla** — o essencial: manipular o DOM, eventos, e principalmente **`fetch`** pra consumir a API de folha (ver [[APIs REST e Sanctum]]).
- Noção de **Alpine.js** (leve, usado pelo Filament) pra interatividade sem framework pesado — opcional.

## Onde estudar
- **MDN Web Docs** (developer.mozilla.org) — a referência definitiva de HTML/CSS/JS.
- **Blade**: Laravel docs → "Blade Templates".
- **CSS**: "CSS Grid" e "Flexbox" no **CSS-Tricks** (guias visuais); jogo **Flexbox Froggy**.
- **Tailwind**: tailwindcss.com/docs.
- **JavaScript**: **javascript.info** — moderno e completo.

## Como saber que entendeu
Consegue montar a tela `meus-holerites` em Blade, estilizada com Flexbox/Grid, buscando os dados via `fetch` no endpoint de folha e renderizando a lista de lançamentos.
