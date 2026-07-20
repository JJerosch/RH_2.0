# Docker

## O que é
**Docker** empacota a aplicação e tudo que ela precisa (PHP, extensões, Node, Chromium, Postgres) em **containers** que rodam igual em qualquer máquina. **docker-compose** orquestra vários containers juntos.

## Por que neste projeto
- Resolve o "**na minha máquina funciona**": mesmo ambiente no seu PC e no deploy.
- É como o **Chromium** do `spatie/laravel-pdf` (D-09) é instalado sem dor — uma linha no Dockerfile.
- Sobe app + Postgres com um comando. Entra na **Fase 4** do cronograma.

## Fundamentos a dominar
- **Imagem × Container**: imagem é o "molde", container é a instância rodando.
- **Dockerfile** — receita pra construir a imagem da app (base PHP, instalar extensões, Composer, Node/Chromium).
- **docker-compose.yml** — define serviços (app, db), redes, **volumes** (persistir dados do Postgres), variáveis de ambiente.
- **Portas** e **volumes** — expor a app e não perder o banco ao recriar o container.
- **`.env`** e como injetar segredos sem colocá-los na imagem (ver [[Segurança e LGPD]]).
- Comandos: `docker compose up/down`, `build`, `logs`, `exec`.

## Onde estudar
- **Documentação oficial** (docs.docker.com) → "Get Started".
- **Laravel Sail** (laravel.com/docs → Sail) — o Docker "de fábrica" do Laravel, ótimo ponto de partida antes de escrever um Dockerfile próprio.
- Busque "**Docker for PHP/Laravel developers**" pra exemplos com Postgres.

## Como saber que entendeu
Consegue subir a aplicação + Postgres com `docker compose up`, entender onde o Chromium é instalado pro PDF, e explicar por que o `.env` não vai dentro da imagem.
