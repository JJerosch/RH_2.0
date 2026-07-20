# Laravel

## O que é
O **framework PHP** que estrutura todo o backend: rotas, banco (via Eloquent), autenticação, e-mail, filas, templates. É o coração do projeto. Pré-requisito: [[PHP e Composer]].

## Por que neste projeto
Praticamente tudo que você vai escrever passa pelo Laravel. Dominá-lo é dominar 60% do projeto.

## Fundamentos a dominar
- **Ciclo de vida da requisição**: como um request entra por `routes/`, passa por middleware, chega no controller.
- **Rotas** (`routes/web.php`, `routes/api.php`) e **controllers**.
- **Eloquent ORM** — o mais importante: models, relações (`hasMany`, `belongsTo`), query builder, `casts`. É como o PHP conversa com o Postgres sem escrever SQL na mão.
- **Migrations** — versionar o schema do banco em código (substituem o Flyway do projeto Java).
- **Middleware** — filtros de requisição (ex.: `ExigeDoisFatores` em [[Autenticação e 2FA]]).
- **Service Container / Injeção de Dependência** — como o Laravel monta objetos com suas dependências. Liga com [[Arquitetura de Software]].
- **Validação** de dados de entrada (Form Requests).
- **Blade** — o motor de templates → [[Frontend (Blade, HTML, CSS, JS)]].
- **Mail** (envio de e-mail — código 2FA, senha inicial) e **Cache** (explicação de holerite cacheada).
- **Eloquent Observers** — usados no projeto pra criar `FeriasPeriodoAquisitivo` ao contratar (sem trigger de banco).

## Onde estudar
- **Documentação oficial** (laravel.com/docs) — excelente, é a principal fonte.
- **Laracasts** (laracasts.com) — "Laravel From Scratch" é o melhor caminho guiado. Vale muito.
- **Laravel Bootcamp** (bootcamp.laravel.com) — projeto guiado curto e oficial.

## Como saber que entendeu
Consegue criar um model + migration + controller + rota + view que lê e grava no banco, e explicar como a injeção de dependência montou seu service. Depois, entende como isso vira a [[Folha de Pagamento]].
