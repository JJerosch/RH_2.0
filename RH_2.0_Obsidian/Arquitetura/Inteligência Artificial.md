# Inteligência Artificial

Domínio `Ia` — o diferencial de portfólio. Centraliza tudo que chama a **Claude API** (Anthropic). Decisões D-08 e D-12 em [[Registro de Decisões]]. Fundamentos técnicos em [[IA com a Claude API]].

## Regra de ouro (D-12)

> A IA **nunca decide valor monetário** e **nunca persiste dado sem revisão humana.**

Isso é o que separa "diferencial legal" de "risco jurídico". Toda feature de IA tem uma etapa humana obrigatória.

## Cliente: `ClaudeClient` (wrapper Guzzle)

Não existe SDK oficial em PHP, então é um wrapper fino sobre Guzzle contra `https://api.anthropic.com/v1/messages`. Headers: `x-api-key` (do `.env`, nunca hardcoded), `anthropic-version`. Camada mínima de confiabilidade (D-08): timeout + retry com backoff em 429/5xx/529, máx. 3 tentativas.

## Feature 1 — Explicar holerite

- Recebe uma `FolhaPagamento` **já FECHADA** + seus `LancamentoFolha`. A IA **só explica** números que o `ProcessadorFolha` já calculou (ver [[Folha de Pagamento]]).
- Prompt restrito: passa os lançamentos como dado estruturado, com instrução explícita de **não declarar nenhum valor fora da lista recebida**.
- Endpoint `POST /api/folha/{id}/explicar` (só folha fechada). Resposta **cacheada** (folha fechada não muda).
- Botão "Explicar com IA" na tela `meus-holerites`.

## Feature 2 — Extrair dados de currículo

1. Upload de PDF/DOCX na candidatura.
2. `smalot/pdfparser` (PDF) / `PhpOffice/PhpWord` (DOCX) extraem o texto bruto.
3. Texto **sanitizado** antes do prompt (remove padrões de instrução embutida → mitiga prompt injection de currículo malicioso).
4. Claude extrai **JSON estruturado** via tool use / structured output — não texto livre parseado na mão. Formato padrão abaixo (D-31), igual pra toda vaga.
5. **Tela de revisão obrigatória**: RH confere/corrige antes de `Candidato`+`Candidatura` serem salvos.
6. Falha ou baixa confiança → cai pro formulário manual.

### Formato padrão da extração (D-31 — independente da vaga)

| Bloco | Campos |
|---|---|
| **Dados pessoais** | nome, e-mail, telefone, cidade, UF, LinkedIn/portfólio |
| **Resumo/objetivo profissional** | o parágrafo de topo do currículo |
| **Formação** (lista) | curso, instituição, nível (técnico/graduação/pós…), ano de conclusão (ou "cursando") |
| **Experiência profissional** (lista) | empresa, cargo, período (início–fim), resumo das atividades |
| **Competências** | lista de habilidades/tecnologias |
| **Certificações** | lista |
| **Idiomas** | idioma + nível |
| **Pretensão salarial** | valor (se informado) |
| **Disponibilidade / mobilidade** | ex.: imediata, aviso prévio, aceita mudança |

Campos ausentes no currículo ficam vazios (nunca inventados — D-12). Tudo passa pela revisão do RH antes de persistir.

`ExtracaoCurriculo` guarda `dados_extraidos` (coluna `json` do Postgres) + `confirmado_pelo_rh`. O arquivo original vai pro `Storage` (fora do banco).

## Segurança específica
- IA nunca recalcula/decide valores.
- Arquivo: limite de tamanho (~5MB), validação por **magic bytes** (não só extensão), texto tratado como entrada não confiável.
- Toda criação via IA passa por confirmação humana.
- Chave em `.env`. Ver [[Segurança e LGPD]].
