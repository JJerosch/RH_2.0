# IA com a Claude API

## O que é
Como integrar a **API da Claude** (Anthropic) ao projeto pra as duas features de IA. Como **não existe SDK oficial em PHP** (D-08), a comunicação é HTTP direto (Guzzle). Detalhe de arquitetura em [[Inteligência Artificial]].

## Por que neste projeto
É o **diferencial de portfólio**: explicar holerite em linguagem natural e extrair dados de currículo. Mas com a regra de ouro — **IA nunca decide valor, nunca persiste sem revisão humana** (D-12).

## Fundamentos a dominar
- **O que é uma LLM e como uma API de chat funciona**: você manda mensagens (`role`, `content`), recebe uma resposta. Sem estado — cada chamada é independente.
- **HTTP client (Guzzle)**: montar request com headers (`x-api-key`, `anthropic-version`), enviar JSON, ler a resposta.
- **Anatomia do request**: `model`, `max_tokens`, `messages`, `system` prompt, `tools`.
- **Prompt engineering básico**: instruções claras e restritivas (ex.: "não declare nenhum valor fora da lista recebida").
- **Tool use / structured output**: pedir **JSON estruturado** (dados do currículo) em vez de texto livre parseado na mão.
- **Tratamento de erro e confiabilidade**: timeout, retry com backoff em 429 (rate limit) / 529 (overloaded) / 5xx — a "camada mínima" da D-08.
- **Segurança**: chave em `.env`; **prompt injection** — texto de currículo é entrada não confiável, sanitizar antes (ver [[Segurança e LGPD]]).
- **Caching**: a explicação de holerite fechado é cacheada (não muda).

## Onde estudar
- **Documentação da Anthropic** (docs.anthropic.com) — Messages API, tool use, prompt engineering. Fonte principal.
- **Guzzle docs** (docs.guzzlephp.org) — o HTTP client.
- Conceitos de **prompt injection**: material da OWASP sobre LLMs (OWASP Top 10 for LLM Applications).

## Como saber que entendeu
Consegue explicar por que a IA **só explica** um holerite já calculado (nunca recalcula), como pedir JSON estruturado do currículo, e por que toda extração passa por revisão humana antes de salvar.
