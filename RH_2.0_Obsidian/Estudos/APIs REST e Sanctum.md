# APIs REST e Sanctum

## O que é
**API REST** = jeito padrão de expor dados por HTTP (verbos GET/POST/PUT/DELETE, respostas em JSON, status codes). **Laravel Sanctum** = autenticação por token pra essas APIs.

## Por que neste projeto
Só a **Folha** expõe API REST (decisão D-14), porque as telas Blade do funcionário consomem esses dados via `fetch`. Os 6 endpoints de folha (holerite, PDF, explicar por IA) estão em [[Folha de Pagamento]].

## Fundamentos a dominar
- **Verbos HTTP** e o que cada um significa (GET lê, POST cria/age).
- **Status codes**: 200 (ok), 201 (criado), 403 (proibido — usado na reabertura por hierarquia), 404, 409 (conflito — folha já fechada), 422 (validação), 500.
- **JSON** como formato de request/response.
- **API Resources** do Laravel — transformar models em JSON de forma controlada.
- **Autenticação dupla** do projeto: sessão (guard `web`) pras telas Blade, **Sanctum** (token) pra API.
- **Idempotência e verbos seguros** — por que "processar folha" é POST, não GET.
- Noção de **CORS** (se algum front externo consumir).

## Onde estudar
- **Laravel docs → "Sanctum"** e **"Eloquent: API Resources"**.
- **MDN Web Docs → HTTP** (developer.mozilla.org) — a melhor referência de status codes e verbos.
- Busque "**REST API design best practices**" pra convenções de nomes de rota.

## Como saber que entendeu
Consegue explicar por que `POST /api/folha/{id}/reabrir` devolve 403 pra quem não é ADMIN/DP_CHEFE, e por que reprocessar folha fechada devolve 409. Liga com [[Perfis de Acesso]] e [[Segurança e LGPD]].
