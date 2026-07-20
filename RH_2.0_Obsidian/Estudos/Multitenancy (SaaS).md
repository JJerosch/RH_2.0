# Multitenancy (SaaS)

## O que é
**Multitenancy** = um único sistema serve **várias empresas (tenants)** ao mesmo tempo, com os dados de cada uma **isolados** das outras. É o "S" de SaaS (Software as a Service). Decisão de arquitetura ligada à D-15.

## Por que neste projeto
O RHParaTodos é um **SaaS multitenant** (Q1): cada empresa cliente tem seus funcionários, folhas e configurações, e ninguém enxerga o dado do outro. O e-mail gerado na contratação já usa a "tag da empresa" (`@{tag}.com.br`). Vários parâmetros são **por empresa** (tolerância de ponto D-24, período aquisitivo D-27).

## Fundamentos a dominar
- **O conceito de tenant** e por que isolamento é questão de **segurança**, não só organização (vazar dado entre empresas é gravíssimo — ver [[Segurança e LGPD]]).
- **Estratégias de isolamento** (entender os trade-offs):
  - **Single database, coluna `tenant_id`** — uma coluna em cada tabela + filtro global. Mais simples, o mais comum. Provável escolha aqui.
  - **Schema por tenant** / **banco por tenant** — mais isolamento, mais complexidade operacional.
- **Global Scopes do Eloquent** — filtrar automaticamente tudo pelo tenant atual, pra nunca esquecer o `where tenant_id = ?`.
- **Identificação do tenant** — por subdomínio (`empresa.rhparatodos.com`), por domínio, ou por usuário logado.
- **Configuração por tenant** — parâmetros que variam por empresa (tolerância, aquisitivo).
- Como isso interage com **cache, filas e jobs** (o job precisa saber de qual tenant é) — liga com [[Filas e Agendamento]].

## Onde estudar
- Pacote **`stancl/tenancy`** (tenancyforlaravel.com) — a referência de multitenancy em Laravel, com ótima documentação conceitual mesmo que você não use o pacote.
- Pacote **`spatie/laravel-multitenancy`** — alternativa mais enxuta, docs didáticas.
- Laravel docs → **"Eloquent: Global Scopes"**.
- Busque "**multi-tenant SaaS architecture patterns**" pra os trade-offs de isolamento.

## Como saber que entendeu
Consegue explicar como garantir que uma consulta de funcionários **nunca** retorne gente de outra empresa (global scope por `tenant_id`), e por que esquecer esse filtro é um bug de segurança, não de lógica.
