# Docker — Guia Prático (passo a passo)

Guia de mão pra quem entende o **conceito** de Docker mas nunca usou na prática. Foco no que ESTE projeto precisa (Laravel + Postgres, e depois Chromium). A teoria/fundamentos está em [[Docker]].

> [!tip] O atalho pro Sprint 1
> Não precisa escrever `Dockerfile` pra começar. O **Laravel Sail** (nível 3) te dá PHP + Postgres em Docker com um comando. Comece por ele; os níveis 1–2 são só pra entender o que o Sail faz por baixo.

---

## Modelo mental (relembrando, 2 min)

- **Imagem** = molde congelado (ex.: "PHP 8.3 + extensões"). Você baixa ou constrói.
- **Container** = uma imagem *rodando*. Descartável — pode destruir e recriar à vontade.
- **Volume** = pasta que **persiste** fora do container (pra não perder os dados do Postgres ao recriar).
- **Porta** = como você acessa o container de fora (ex.: `localhost:5432` → Postgres).
- **docker-compose** = um arquivo que sobe **vários** containers juntos (app + banco) com um comando.

---

## Nível 0 — Instalar (Windows)

1. Instale o **Docker Desktop** (docker.com → Download). Ele usa o **WSL2** no Windows — se pedir, ative o WSL2 (o instalador guia).
2. Abra o Docker Desktop e espere ficar "Engine running".
3. Teste no terminal:
   ```bash
   docker run hello-world
   ```
   Se imprimir uma mensagem de boas-vindas, está funcionando.

**O que praticar aqui:** só rodar `hello-world` e `docker ps` (lista containers ativos). Nada além disso.

---

## Nível 1 — Rodar um container qualquer

```bash
docker run -it --rm ubuntu bash    # entra num Ubuntu descartável
# digite: ls, whoami, exit
```
- `--rm` = apaga o container ao sair (descartável).
- `-it` = modo interativo.

**Objetivo:** sentir que um container é um "computador de mentira" isolado. Saiu, sumiu.

---

## Nível 2 — Rodar um Postgres (o banco do projeto)

```bash
docker run --name pg-teste \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=rh \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:16
```
- `-e` = variável de ambiente (senha, nome do banco).
- `-p 5432:5432` = expõe a porta do Postgres pra sua máquina.
- `-v pgdata:/var/lib/...` = **volume**: os dados sobrevivem mesmo se você destruir o container.
- `-d` = roda em segundo plano.

Comandos de apoio:
```bash
docker ps                 # ver rodando
docker logs pg-teste      # ver o log
docker stop pg-teste      # parar
docker rm pg-teste        # remover (dados ficam no volume pgdata)
```

**Objetivo:** entender env vars, porta e volume — os 3 conceitos que o Sail/compose usam.

---

## Nível 3 — Laravel Sail (é por aqui que o projeto começa) ⭐

O Sail é um `docker-compose` pronto do Laravel. Ao criar o projeto:

```bash
# cria o projeto já com Sail + Postgres
curl -s "https://laravel.build/rhparatodos?with=pgsql" | bash
cd rhparatodos

# sobe tudo (app PHP + Postgres) em Docker
./vendor/bin/sail up -d
```

Depois é só usar o Sail no lugar dos comandos locais:
```bash
./vendor/bin/sail artisan migrate      # em vez de "php artisan migrate"
./vendor/bin/sail artisan test         # roda o Pest
./vendor/bin/sail composer require ...  # composer dentro do container
./vendor/bin/sail down                 # desliga
```

Dá pra criar um atalho `sail` no terminal pra não digitar o caminho todo (a doc mostra).

**Objetivo:** ter o ambiente do Sprint 1 rodando sem escrever Dockerfile. Isso já é "Docker desde o início".

---

## Nível 4 — Entender o `docker-compose.yml` do Sail

O Sail gera um `docker-compose.yml` na raiz. **Abra e leia** — você vai reconhecer tudo dos níveis 2–3: o serviço `laravel.test` (a app), o serviço `pgsql` (banco), as portas, os volumes, a rede. Mexer aqui é como você vai, mais pra frente, **adicionar o Chromium** pro PDF (D-09) e ajustar pro deploy.

**Objetivo:** deixar de ver o compose como caixa-preta.

---

## Nível 5 (Fase 4) — Dockerfile próprio + Chromium

Só quando for pra deploy (Fase 4 do plano). Aí você escreve um `Dockerfile` próprio (base PHP-FPM + Nginx), instala as extensões e o **Chromium** que o `spatie/laravel-pdf` precisa. Não é Sprint 1 — anota e segue.

---

## Colinha de comandos

| Comando | O quê |
|---|---|
| `docker ps` | containers rodando |
| `docker images` | imagens baixadas |
| `sail up -d` / `sail down` | liga / desliga o projeto |
| `sail artisan <cmd>` | roda artisan no container |
| `sail test` | roda os testes |
| `docker system prune` | limpa lixo (containers/imagens parados) |

## Onde aprofundar
- **Laravel Sail** — laravel.com/docs → "Sail" (a fonte pro nosso caso).
- **Docker Desktop / Get Started** — docs.docker.com.
- Fundamentos e trade-offs: [[Docker]].

## Como saber que entendeu (pro Sprint 1)
Consegue: subir o projeto com `sail up -d`, rodar uma migration e os testes pelo Sail, parar e subir de novo **sem perder os dados** do Postgres, e apontar no `docker-compose.yml` quais são os serviços da app e do banco.
