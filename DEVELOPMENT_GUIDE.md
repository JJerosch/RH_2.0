# RHParaTodos-Django — Guia de Desenvolvimento

> Este documento traduz as decisões do `RHParaTodos-Django-plano.md` (plano técnico/arquitetural, não alterado) em um guia prático de implementação. Todo o conteúdo abaixo deriva exclusivamente do que está escrito nesse plano. Onde o plano não tem detalhe suficiente, isso é marcado explicitamente com:
>
> ⚠️ **Lacuna no plano** — descrição do que falta.
>
> Esse marcador não é um defeito deste guia — é um inventário honesto do que ainda precisa ser decidido antes ou durante a implementação.

---

# 1. Visão Geral do Produto

**O que é o RHParaTodos**: um sistema de RH (Recursos Humanos) e DP (Departamento Pessoal), atualmente em reconstrução como `RHParaTodos-Django`. É a reescrita de um projeto anterior em Java/Spring Boot que ficou incompleto, com a regra de cálculo de folha de pagamento implementada apenas em functions/triggers PL/pgSQL nunca chamadas pela aplicação, telas de holerite 100% simuladas (`Math.random()`/`localStorage`), rotas de segurança apontando para módulos nunca implementados, contratação automática com senha previsível, e segredos reais commitados no repositório git.

**Público-alvo**: ⚠️ **Lacuna no plano** — o documento não define formalmente quem usa o sistema (porte de empresa, número de funcionários esperado, setor). O que é possível inferir apenas dos perfis de acesso citados (`ADMIN`, `DP_CHEFE` mencionados na tabela de endpoints de folha) é que existem pelo menos dois grupos: equipe interna de RH/DP (administra folha, recrutamento, cadastros) e funcionários comuns (consultam o próprio holerite, recebem explicação por IA). Não há definição de escala (pequena/média empresa) nem de mercado.

**Problemas que resolve** (extraído da seção "Contexto" do plano, 6 falhas do projeto original que motivam a reconstrução):
1. Cálculo de holerite (INSS/IRRF/FGTS) existindo só no banco, nunca chamado pela aplicação.
2. Tela de holerites 100% simulada, sem nenhuma chamada de API real.
3. Rota `/payslips` quebrada (view inexistente).
4. Configuração de segurança referenciando módulos (`/payroll`, `/reports`, `/performance`, `/training`) que nunca foram implementados.
5. Contratação automática gerando senha previsível e não inicializando férias do novo funcionário.
6. Segredos reais (senha de e-mail, chave JWT) commitados no controle de versão.

A auditoria documentada no plano identificou ainda 4 problemas adicionais (não listados nos 6 originais, mas registrados na seção "Correções específicas"): desligamento sem cálculo de rescisão financeira, bug de schema no 2FA (hash truncado), ausência de rastreabilidade entre vaga e funcionário contratado, e módulos de ponto/férias completamente desconectados da folha.

**Objetivos do sistema** (conforme o plano):
- Reconstruir o sistema em **Django + DRF**, mantendo toda regra de negócio de folha de pagamento (e, por extensão, rescisão) em **código Python testável**, nunca em função/trigger de banco.
- Adicionar dois diferenciais de inteligência artificial (API Claude) como diferencial de portfólio: guia de holerites em linguagem natural e extração automática de dados de currículo.
- Seguir uma regra de processo explícita ("Definition of Done") para que nenhuma rota/funcionalidade fique "meio pronta", repetindo o erro #4 do projeto original.
- ⚠️ **Lacuna no plano**: não há uma declaração formal de objetivo de negócio (ex.: "reduzir tempo de processamento de folha em X%", "atender N funcionários"). Os objetivos documentados são todos objetivos de **engenharia/arquitetura**, não de produto/negócio.

---

# 2. Escopo do MVP

O plano não usa a expressão "MVP" como um corte formal único — ela aparece uma vez, na tabela de correções ("App `relatorios` entra no MVP da Fase 2"). Não existe uma lista explícita e centralizada de "isso é MVP, isso não é". O escopo abaixo é reconstruído a partir do cronograma (todas as fases 0–5 são tratadas como entregáveis do plano corrente, não como "futuro") e das exclusões explícitas mencionadas em diferentes pontos do documento.

## Funcionalidades obrigatórias (presentes no cronograma, fases 0–5)

| Fase | Entregável (conforme plano) |
|---|---|
| 0 — Setup + 2FA | Projeto Django, app `core` (auth customizada, perfis, 2FA por e-mail corrigido), app `organizacao`, `.env`, CI básico com `pytest` |
| 1 — Domínio base | Apps `funcionarios`, `ponto`, `ferias`, `beneficios` com CRUD via DRF + Django Admin |
| 2 — Núcleo de folha | App `folha` completo: models, calculadoras de INSS/IRRF/FGTS, `processador_folha`, integração ponto/férias, testes, endpoints de holerite consumidos pelo front |
| 2.5 — Rescisão | Cálculo financeiro completo de desligamento (saldo, férias proporcionais, 13º proporcional, aviso prévio, multa FGTS) |
| 3 — Recrutamento + Relatórios | Fluxo de contratação corrigido (senha segura, sinal de férias, rastreabilidade vaga→funcionário) + 2-3 relatórios reais |
| 4 — Docker + ajustes | `docker-compose` (Django + Postgres), checklist OWASP básico, polimento de front |
| 5 — IA (diferencial) | Guia de holerites por IA + extração de currículo com upload/parsing/revisão humana |

## Funcionalidades explicitamente fora de escopo por agora

- **React no frontend** — decisão explícita do plano ("sem React por agora"), front fica em templates Django + JS vanilla.
- **README comparativo, dados de demo (seed/fixtures) e deploy** — citados como "fora deste documento por ora — entram quando você pedir" (cronograma, nota final).
- **Laravel** — citado apenas como possibilidade de uso da margem de tempo ("ou avançar para Laravel... se tudo estiver entregue antes"), não como compromisso.
- **Congelamento/armazenamento do PDF do holerite** — decisão explícita de gerar dinamicamente; armazenar ficou marcado como "não necessário agora", a revisitar só se o projeto virar produção real.

## ⚠️ Lacunas no plano sobre escopo

- Não há critério explícito de "o que é opcional dentro de uma fase" — cada fase é tratada como bloco indivisível no cronograma.
- Não há priorização entre os 9 apps listados na estrutura além da ordem do cronograma (ex.: não está definido se `beneficios` pode ser cortado/simplificado sem afetar `folha`).
- O escopo de `relatorios` é dado apenas como exemplo ("2-3 relatórios simples: headcount por departamento, custo de folha por mês, férias vencendo") — não há lista fechada e definitiva.

---

# 3. Personas

O plano não define personas (perfis com nome, rotina, objetivos, dores). As únicas informações de "quem faz o quê" vêm de menções pontuais de perfil de acesso espalhadas pelo documento (ex.: "Só ADMIN/DP_CHEFE" para reabrir folha; "RH vê o que a IA extraiu" na extração de currículo; "tela de holerite do funcionário"). As 4 personas abaixo são construídas **apenas com essas menções pontuais** — qualquer detalhe além disso (idade, rotina diária, formação) seria invenção e foi evitado.

### Administrador RH/DP
- **Evidência no plano**: perfis `ADMIN` e `DP_CHEFE` citados como os únicos que podem reabrir uma folha fechada; "RH/DP" citado como quem processa a folha.
- **Objetivo inferido**: processar, fechar e (quando necessário) reabrir folhas de pagamento; revisar e confirmar dados extraídos por IA antes de criar candidatos.
- ⚠️ **Lacuna no plano**: não há lista fechada de perfis (o plano cita `ADMIN`, `DP_CHEFE`, "RH/DP", "RH" — sem esclarecer se são o mesmo perfil, perfis distintos, ou um subconjunto de uma lista maior).

### Gestor
- ⚠️ **Lacuna no plano**: a palavra "Gestor" **não aparece em nenhum lugar do plano**. Não há nenhum endpoint, regra de negócio ou menção de acesso associada a um perfil com esse nome. Esta persona é solicitada pela estrutura deste guia, mas não tem nenhuma base no documento fonte — precisa ser definida com você antes de qualquer requisito ser escrito em nome dela.

### Funcionário
- **Evidência no plano**: "tela de holerite do funcionário (`meus-holerites`)"; endpoint `GET /api/folha/meus-holerites/`; botão "Explicar com IA" na própria tela de holerite.
- **Objetivo inferido**: consultar o próprio holerite, baixar o PDF, pedir uma explicação em linguagem natural sobre os valores descontados.
- ⚠️ **Lacuna no plano**: nenhuma outra interação do funcionário (ex.: solicitar férias, bater ponto) é descrita do ponto de vista de tela/fluxo — só os modelos de dados (`FeriasSolicitacao`, `PontoMarcacao`) são citados.

### Candidato
- **Evidência no plano**: upload de currículo (PDF/DOCX) "na tela pública/RH de candidatura"; dados extraídos por IA (nome, email, telefone, cidade, estado, resumo de experiência).
- **Objetivo inferido**: se candidatar a uma vaga enviando currículo, sem precisar preencher manualmente todos os campos (a IA pré-extrai os dados, mas a confirmação final é do RH, não do candidato).
- ⚠️ **Lacuna no plano**: não está definido se o candidato tem alguma conta/login no sistema ou se a candidatura é 100% anônima/pública até a etapa de revisão do RH.

---

# 4. Requisitos Funcionais

Cada item abaixo cita a origem no plano. Itens marcados ⚠️ existem apenas como nome de modelo na "Estrutura de apps", sem fluxo, campo ou regra descrita.

| ID | Requisito | Origem no plano |
|---|---|---|
| RF-001 | Autenticar usuário (e-mail/senha) | Seção "Stack" — JWT (`simplejwt`) + sessão Django |
| RF-002 | Gerar código de 2FA por e-mail no login | Seção "App `core`" — `gerar_codigo` |
| RF-003 | Validar código de 2FA informado pelo usuário | Seção "App `core`" — `validar_codigo` |
| RF-004 | Bloquear tentativas de 2FA após N erros | Seção "App `core`" — rate limiting básico |
| RF-005 | Bloquear acesso a rotas até 2FA verificado | Seção "App `core`" — middleware |
| RF-006 | ⚠️ Cadastrar Funcionário | App `funcionarios` citado na estrutura — sem campos/fluxo |
| RF-007 | ⚠️ Registrar Promoção de funcionário | `Promocao` citado na estrutura — sem campos/fluxo |
| RF-008 | ⚠️ Cadastrar Departamento | App `organizacao` — sem campos/fluxo |
| RF-009 | ⚠️ Cadastrar Cargo | App `organizacao` — sem campos/fluxo |
| RF-010 | ⚠️ Cadastrar Vaga | App `recrutamento` — sem campos/fluxo |
| RF-011 | ⚠️ Cadastrar Candidato (manual) | App `recrutamento` — sem campos/fluxo manual |
| RF-012 | ⚠️ Registrar Candidatura (candidato↔vaga) | App `recrutamento` — sem fluxo |
| RF-013 | ⚠️ Mover candidatura entre etapas | Não descrito — só citado indiretamente via correções |
| RF-014 | Contratar candidato com senha segura gerada | "Correções específicas" — `secrets.token_urlsafe` |
| RF-015 | Forçar troca de senha no primeiro login | "Correções específicas" — `must_change_password` |
| RF-016 | Inicializar período aquisitivo de férias na contratação | "Correções específicas" — signal em `recrutamento` |
| RF-017 | Registrar vaga de origem no funcionário contratado | "Correções específicas" — `Funcionario.vaga_origem` |
| RF-018 | Upload de currículo (PDF/DOCX) | App `ia`, seção 2 |
| RF-019 | Extrair texto de currículo (PDF/DOCX) | App `ia` — `pdfplumber`/`python-docx` |
| RF-020 | Sanitizar texto extraído contra prompt injection | App `ia` — seção "Segurança específica da IA" |
| RF-021 | Extrair dados estruturados de currículo via IA | App `ia` — `extrator_curriculo.py` |
| RF-022 | Revisar/confirmar dados extraídos antes de persistir | App `ia` — "Tela de revisão obrigatória" |
| RF-023 | Cair para cadastro manual se extração falhar | App `ia` — fallback explícito |
| RF-024 | Gerar PDF de perfil do candidato | App `ia`, seção 2 — botão "Ver detalhes" |
| RF-025 | ⚠️ Registrar marcação de ponto | `PontoMarcacao` citado — sem fluxo |
| RF-026 | ⚠️ Apurar ponto diário (horas/extra/falta) | `PontoApuracaoDiaria` citado — consumido por `gerar_lancamentos_ponto`, mas apuração em si não descrita |
| RF-027 | ⚠️ Registrar ocorrência de ponto | `PontoOcorrencia` citado — sem fluxo |
| RF-028 | ⚠️ Gerenciar calendário de ponto | `PontoCalendario` citado — sem fluxo |
| RF-029 | ⚠️ Solicitar férias | `FeriasSolicitacao` citado — sem fluxo de aprovação |
| RF-030 | Registrar férias aprovadas para gerar lançamento de folha | `FeriasRegistro` usado por `gerar_lancamentos_ferias` |
| RF-031 | ⚠️ Gerenciar período aquisitivo de férias | `FeriasPeriodoAquisitivo` citado — sem regra de cálculo de período |
| RF-032 | ⚠️ Cadastrar tipo de benefício | `TipoBeneficio` citado — sem campos/fluxo |
| RF-033 | ⚠️ Vincular benefício a funcionário | `FuncionarioBeneficio` citado — sem campos/fluxo |
| RF-034 | Abrir folha de pagamento | App `folha` — `FolhaPagamento`, status `ABERTA` |
| RF-035 | Gerar lançamentos de salário base na folha | `processador_folha` |
| RF-036 | Gerar lançamentos de ponto (horas extra/falta) na folha | `gerar_lancamentos_ponto` |
| RF-037 | Gerar lançamentos de férias na folha | `gerar_lancamentos_ferias` |
| RF-038 | Calcular INSS progressivo | `calculadora_inss.py` |
| RF-039 | Calcular IRRF progressivo | `calculadora_irrf.py` |
| RF-040 | Calcular FGTS | `calculadora_fgts.py` |
| RF-041 | Processar folha (orquestração completa) | `processador_folha.py` |
| RF-042 | Fechar folha (bloquear novos lançamentos) | `FolhaJaFechadaError` |
| RF-043 | Reabrir folha (somente ADMIN/DP_CHEFE) | Endpoint `POST /api/folha/<id>/reabrir/` |
| RF-044 | Consultar holerites do próprio funcionário | `GET /api/folha/meus-holerites/` |
| RF-045 | Gerar PDF do holerite dinamicamente | Endpoint `GET /api/folha/<id>/pdf/` |
| RF-046 | Explicar holerite em linguagem natural via IA | Endpoint `POST /api/folha/<id>/explicar/` |
| RF-047 | Registrar tipo e motivo de desligamento | App `folha`/rescisão — `tipo_desligamento`, `aviso_previo_tipo` |
| RF-048 | Calcular saldo de salário na rescisão | `calculadora_saldo_salario.py` |
| RF-049 | Calcular férias rescisórias (vencidas + proporcionais) | `calculadora_ferias_rescisao.py` |
| RF-050 | Calcular 13º proporcional na rescisão | `calculadora_decimo_terceiro_proporcional.py` |
| RF-051 | Calcular aviso prévio (indenizado/trabalhado/dispensado) | `calculadora_aviso_previo.py` |
| RF-052 | Calcular multa de FGTS conforme tipo de desligamento | `calculadora_multa_fgts.py` |
| RF-053 | Aplicar regras de verbas por tipo de desligamento | `regras_por_tipo_desligamento.py` |
| RF-054 | Gerar folha de rescisão e fechar automaticamente | `processador_rescisao.py` |
| RF-055 | ⚠️ Gerenciar Solicitação genérica (RH) | App `solicitacoes` (`Solicitacao`, `StatusSolicitacao`, `TipoSolicitacao`) citado — sem nenhum fluxo |
| RF-056 | Gerar relatório de headcount por departamento | App `relatorios` |
| RF-057 | Gerar relatório de custo de folha por mês | App `relatorios` |
| RF-058 | Gerar relatório de férias vencendo | App `relatorios` |

---

# 5. Requisitos Não Funcionais

| ID | Categoria | O que o plano diz | Lacunas |
|---|---|---|---|
| RNF-001 | Segurança | Segredos via `.env`/`django-environ`, nunca hardcoded; senha de contratação aleatória + troca obrigatória; 2FA por e-mail; upload de currículo com limite de tamanho (5MB) e validação de tipo real (magic bytes); texto de IA sanitizado contra prompt injection; toda criação via IA passa por confirmação humana; "checklist OWASP básico" na Fase 4 | ⚠️ O plano não lista quais itens do OWASP entram nesse checklist |
| RNF-002 | Performance | Não mencionado | ⚠️ Nenhum requisito de tempo de resposta, throughput ou SLA definido em nenhum ponto do plano |
| RNF-003 | Auditoria | Não mencionado | ⚠️ O projeto original tinha um trigger `fn_auditoria_lgpd` (citado na auditoria, não no plano novo). O plano não propõe nenhum substituto em Python — gap real, já que o sistema trata CPF e salário |
| RNF-004 | Logs | Não mencionado | ⚠️ Nenhuma estratégia de logging estruturado, nível de log, ou retenção é definida |
| RNF-005 | Disponibilidade | Não mencionado | ⚠️ Sem definição de uptime, ambiente de hospedagem (o próprio cronograma exclui deploy "por ora") |
| RNF-006 | Escalabilidade | Não mencionado | ⚠️ Projeto dimensionado implicitamente para um único desenvolvedor/portfólio, não há discussão de carga, número de usuários simultâneos ou particionamento |
| RNF-007 | Testabilidade | Bem coberto: `pytest-django`; services de folha/rescisão/IA/2FA desenhados para serem testáveis sem depender de trigger de banco; arquivos de teste nomeados explicitamente para folha, rescisão e IA | — |
| RNF-008 | Manutenibilidade/Processo | "Definition of Done" formal (5 condições) impede feature "meio pronta" no `main`; regra de que nenhuma rota entra em `urls.py` sem tela funcionando atrás | — |

---

# 6. Casos de Uso por Módulo

## Core (autenticação e 2FA)
- **Objetivo**: autenticar o usuário com segurança, exigindo um segundo fator por e-mail antes de liberar acesso.
- **Ator**: qualquer usuário do sistema (perfil não diferenciado nesta etapa).
- **Fluxo principal**: usuário envia e-mail/senha → sistema autentica (JWT + sessão) → `gerar_codigo` cria um código de 6 dígitos, salva o hash (`CodigoDoisFatores`) e envia por e-mail → usuário informa o código → `validar_codigo` confirma → middleware libera acesso às rotas protegidas.
- **Fluxos alternativos**: ⚠️ **Lacuna no plano** — o que acontece se o usuário não receber o e-mail e pedir reenvio não é descrito (a única operação descrita é invalidar códigos anteriores ao gerar um novo, o que sugere que reenviar = gerar de novo, mas isso não está explícito).
- **Exceções**: código incorreto incrementa `tentativas`; após N tentativas o sistema bloqueia (⚠️ valor de N não definido no plano); código expirado não é aceito (⚠️ tempo de expiração citado só como "ex.: 5 min", não fixado como requisito).

## Funcionários
- **Objetivo**: ⚠️ **Lacuna no plano** — não descrito além do nome do app e dos modelos (`Funcionario`, `Promocao`).
- **Ator**: ⚠️ não especificado (presumivelmente RH/DP, por analogia com os outros módulos administrativos).
- **Fluxo principal**: ⚠️ **Lacuna no plano** — não há fluxo de cadastro manual de funcionário descrito. O único fluxo de criação de `Funcionario` documentado no plano é via contratação (módulo Recrutamento), não via cadastro direto neste módulo.
- **Fluxos alternativos / Exceções**: ⚠️ não descritos.
- **Observação**: os campos novos `tipo_desligamento`, `aviso_previo_tipo` e `vaga_origem` (adicionados a `Funcionario` pelas seções de Rescisão e Correções) são a única extensão de modelo descrita para este módulo no plano.

## Recrutamento
- **Objetivo**: ⚠️ **Lacuna no plano** — objetivo do módulo em si não é narrado; só são descritas 3 correções pontuais aplicadas "na contratação".
- **Ator**: candidato (envia currículo), RH (revisa/confirma dados extraídos e, presumivelmente, conduz a contratação).
- **Fluxo principal**: ⚠️ **Lacuna no plano** — não há descrição de como uma `Vaga` é criada, como um `Candidato` se torna uma `Candidatura`, nem quais etapas uma candidatura percorre. O plano só documenta o que acontece **no momento da contratação**: gera senha aleatória + força troca no primeiro login, dispara signal que cria `FeriasPeriodoAquisitivo`, e grava `Funcionario.vaga_origem`.
- **Fluxos alternativos**: o fluxo de extração de currículo por IA (ver módulo IA) alimenta este módulo, mas a integração entre "dados extraídos" e "candidatura criada" não tem o trigger exato descrito (ex.: é automático após confirmação do RH, ou precisa de uma ação extra?). ⚠️ Lacuna.
- **Exceções**: ⚠️ não descritas (ex.: o que ocorre se o CPF do candidato já existir como funcionário — não mencionado no plano novo, embora fosse uma regra do projeto original).

## Ponto
- **Objetivo**: ⚠️ **Lacuna no plano** — apenas os modelos são citados na estrutura de apps (`PontoMarcacao`, `PontoApuracaoDiaria`, `PontoCalendario`, `PontoOcorrencia`).
- **Ator**: ⚠️ não especificado.
- **Fluxo principal**: ⚠️ **Lacuna no plano** quase total. A única operação descrita que toca este módulo é o consumo, pela folha, de `PontoApuracaoDiaria` já apurado: `gerar_lancamentos_ponto(folha, competencia)` "lê `PontoApuracaoDiaria` do período e cria lançamentos HEXTRA/FALTA". Como a marcação se transforma em apuração diária não é descrito.
- **Fluxos alternativos / Exceções**: ⚠️ não descritos.

## Férias
- **Objetivo**: ⚠️ **Lacuna no plano** — apenas os modelos são citados (`FeriasPeriodoAquisitivo`, `FeriasSolicitacao`, `FeriasRegistro`).
- **Ator**: ⚠️ não especificado.
- **Fluxo principal**: dois pontos são descritos com precisão: (1) na contratação, um signal cria `FeriasPeriodoAquisitivo` automaticamente; (2) quando uma `FeriasRegistro` é aprovada para a competência de uma folha aberta, `gerar_lancamentos_ferias(folha, ferias_registro)` cria os lançamentos `FERIAS`/`FERIAS_1_3`. O processo de solicitação (`FeriasSolicitacao`) e sua aprovação não são descritos.
- **Fluxos alternativos / Exceções**: ⚠️ não descritos.

## Benefícios
- **Objetivo**: ⚠️ **Lacuna no plano** — apenas os modelos são citados (`TipoBeneficio`, `FuncionarioBeneficio`).
- **Ator / Fluxo principal / Alternativos / Exceções**: ⚠️ **Lacuna total no plano**. Nenhuma regra, campo ou fluxo é descrito para este módulo além do nome dos dois modelos na árvore de apps.

## Folha
- **Objetivo**: processar a folha de pagamento mensal e a rescisão de forma determinística, testável e auditável, sem nenhuma regra de cálculo no banco.
- **Ator**: RH/DP (processa, fecha, gera lançamentos), ADMIN/DP_CHEFE (reabre folha fechada), Funcionário (consulta holerite e PDF, solicita explicação por IA).
- **Fluxo principal**: folha é aberta (`status="ABERTA"`) → `processador_folha` apaga lançamentos automáticos antigos → calcula base e valor de INSS → calcula base e valor de IRRF (base líquida = base IRRF − INSS) → calcula FGTS (lançamento informativo) → recalcula totais (`total_proventos`, `total_descontos`, `valor_liquido`) → folha pode ser fechada (`fechar_folha`, bloqueando novos lançamentos) → holerite consultável via API e PDF gerado dinamicamente a cada download → funcionário pode pedir explicação por IA dos lançamentos já fechados.
- **Fluxos alternativos**: folha fechada pode ser reaberta (somente ADMIN/DP_CHEFE) para permitir reprocessamento; rescisão segue um fluxo paralelo (`processador_rescisao`) que gera uma `FolhaPagamento` do tipo `RESCISAO`, reaproveitando `calcular_inss`/`calcular_irrf`, mas com calculadoras próprias para saldo, férias rescisórias, 13º proporcional, aviso prévio e multa de FGTS, seguindo uma tabela de regras por tipo de desligamento.
- **Exceções**: tentar reprocessar uma folha com `status="FECHADA"` levanta `FolhaJaFechadaError`.

## Relatórios
- **Objetivo**: oferecer visibilidade gerencial que nunca existiu no projeto original (módulo "esqueletado" no Security mas nunca implementado).
- **Ator**: ⚠️ não especificado (presumivelmente RH/DP, por ser dado gerencial).
- **Fluxo principal**: ⚠️ **Lacuna no plano** — só há 3 exemplos de relatório citados (headcount por departamento, custo de folha por mês, férias vencendo), sem detalhe de filtros, formato de saída (tela, PDF, CSV) ou frequência.
- **Fluxos alternativos / Exceções**: ⚠️ não descritos.

## IA
- **Objetivo**: oferecer dois diferenciais de portfólio — explicação de holerite em linguagem natural e extração automática de dados de currículo — sem que a IA jamais decida valores monetários ou crie registros sem revisão humana.
- **Ator**: Funcionário (pede explicação do próprio holerite), RH (revisa extração de currículo).
- **Fluxo principal (explicação de holerite)**: funcionário aciona "Explicar com IA" na tela `meus-holerites` → endpoint `POST /api/folha/<id>/explicar/` envia os lançamentos já calculados e fechados para o Claude com um prompt restrito → resposta é cacheada (folha fechada não muda).
- **Fluxo principal (extração de currículo)**: candidato/RH faz upload de PDF/DOCX → `pdfplumber`/`python-docx` extraem texto → texto é sanitizado contra prompt injection → Claude extrai JSON estruturado (nome, email, telefone, cidade, estado, resumo) → resultado bruto é salvo em `ExtracaoCurriculo.dados_extraidos` → RH revisa e confirma (`confirmado_pelo_rh`) antes de `Candidato`/`Candidatura` serem persistidos → botão "Ver detalhes" gera PDF de perfil padronizado, mostrando também o dado bruto extraído pela IA como trilha de auditoria.
- **Fluxos alternativos**: se a extração falhar ou tiver confiança baixa, cai para o formulário manual de cadastro (a IA é acelerador, não dependência obrigatória).
- **Exceções**: ⚠️ **Lacuna no plano** — não há descrição do que acontece em caso de erro/timeout da API Claude (rate limit, indisponibilidade) além do fallback geral para cadastro manual no caso específico de currículo; para a explicação de holerite, nenhum comportamento de erro é descrito.

---

# 7. Fluxos de Negócio

### Contratação
1. ⚠️ **Lacuna no plano**: não há descrição de como/quando a contratação é disparada (ex.: mudança de etapa de candidatura). O plano documenta apenas as 3 ações que acontecem **nesse momento**:
2. Gera senha aleatória (`secrets.token_urlsafe`) para o novo usuário e marca `must_change_password=True`.
3. Dispara signal que cria `FeriasPeriodoAquisitivo` para o novo funcionário.
4. Grava `Funcionario.vaga_origem` apontando para a vaga de origem.

### Registro de ponto
⚠️ **Lacuna no plano quase total.** O único passo documentado é o consumo posterior pela folha: `gerar_lancamentos_ponto(folha, competencia)` lê registros já apurados em `PontoApuracaoDiaria` do período e cria lançamentos `HEXTRA`/`FALTA`. Não há descrição de como uma marcação (`PontoMarcacao`) se torna uma apuração diária (`PontoApuracaoDiaria`), nem de regras de jornada, tolerância, ou tratamento de `PontoOcorrencia`/`PontoCalendario`.

### Solicitação de férias
⚠️ **Lacuna no plano sobre a solicitação em si** (`FeriasSolicitacao` — sem fluxo de aprovação descrito). O que o plano documenta é o que acontece **depois** de uma `FeriasRegistro` existir e estar aprovada para a competência de uma folha aberta:
1. `gerar_lancamentos_ferias(folha, ferias_registro)` é chamada explicitamente (por `processador_folha` ou por signal documentado).
2. Cria lançamento `FERIAS` (férias gozadas) e `FERIAS_1_3` (1/3 constitucional).

### Processamento de folha
Este é o fluxo mais detalhado do plano:
1. Folha existe com `status="ABERTA"`.
2. `processador_folha(folha_id)` trava a linha (`select_for_update`) e verifica que não está `FECHADA` (senão levanta `FolhaJaFechadaError`).
3. Remove lançamentos com `origem="CALCULO_AUTOMATICO"` (permite reprocessar do zero).
4. Calcula base de INSS → `calcular_inss(base, ano)` → registra lançamento `DESCONTO`/`INSS`.
5. Calcula base de IRRF (base IRRF menos o INSS já calculado) → `calcular_irrf(base_liquida, data_referencia)` → registra lançamento `DESCONTO`/`IRRF`.
6. Calcula FGTS → `calcular_fgts(base)` → registra lançamento `INFORMATIVO`/`FGTS`.
7. Recalcula totais da folha (`total_proventos`, `total_descontos`, `valor_liquido`).
8. Folha pode ser fechada (`POST /api/folha/<id>/fechar/`), bloqueando novos lançamentos.
9. Folha fechada pode ser reaberta apenas por `ADMIN`/`DP_CHEFE` (`POST /api/folha/<id>/reabrir/`).
10. Holerite consultável (`GET /api/folha/meus-holerites/`, `GET /api/folha/<id>/`) e PDF gerado dinamicamente (`GET /api/folha/<id>/pdf/`) a cada download, nunca armazenado.

### Rescisão
1. `Funcionario` recebe `tipo_desligamento` (`SEM_JUSTA_CAUSA`, `PEDIDO_DEMISSAO`, `JUSTA_CAUSA`, `ACORDO`) e `aviso_previo_tipo` (`INDENIZADO`, `TRABALHADO`, `DISPENSADO`).
2. `processador_rescisao(funcionario_id, tipo_desligamento, data_desligamento, aviso_previo_tipo)` consulta a tabela `REGRAS` para saber quais verbas se aplicam a esse tipo de desligamento.
3. Calcula saldo de salário, férias (vencidas + proporcionais), 13º proporcional e aviso prévio — apenas os que a regra permite.
4. Aplica `calcular_inss`/`calcular_irrf` (reaproveitados da folha mensal) separadamente sobre saldo de salário e sobre 13º.
5. Calcula multa de FGTS sobre o histórico de FGTS depositado (soma dos lançamentos `FGTS` ao longo do vínculo).
6. Gera `FolhaPagamento` do tipo `RESCISAO` com todos os lançamentos e fecha automaticamente (evento único, sem etapa manual de fechamento).

### Recrutamento
⚠️ **Lacuna no plano.** Não há um fluxo de recrutamento ponta a ponta documentado (criação de vaga → candidatura → progressão de etapas → contratação). O plano só descreve o ponto de chegada (contratação, já coberto acima) e o ponto de entrada alternativo via IA (abaixo).

### Extração de currículo por IA
1. Candidato/RH faz upload de PDF/DOCX na tela de candidatura.
2. `pdfplumber` (PDF) ou `python-docx` (DOCX) extraem o texto bruto.
3. Texto é sanitizado antes do prompt (remove padrões de instrução embutida, ex.: "ignore as regras anteriores").
4. Claude extrai JSON estruturado (nome, email, telefone, cidade, estado, resumo de experiência) via structured output/tool use.
5. Resultado bruto é salvo em `ExtracaoCurriculo.dados_extraidos` (JSONField), com `confirmado_pelo_rh=False`.
6. RH revisa na tela de revisão obrigatória, corrige se necessário, e confirma.
7. Só então `Candidato` + `Candidatura` são persistidos — nunca direto da resposta da IA.
8. Se a extração falhar ou tiver confiança baixa, cai para o formulário manual já existente.
9. Botão "Ver detalhes" no Kanban gera PDF de perfil padronizado (`WeasyPrint`), mostrando também o dado bruto da IA como seção secundária/auditoria.

---

# 8. Estrutura de Dados

Entidades agrupadas por app. Entidades com campos completos no plano estão marcadas com ✅; entidades citadas só pelo nome estão marcadas com ⚠️.

## App `core`
- **Usuario** ⚠️ — citado como modelo base de autenticação (`core.Usuario`), referenciado por `FolhaPagamento.fechada_por` e `CodigoDoisFatores.usuario`. Campos além de identidade básica não detalhados. Campo `must_change_password` é mencionado na tabela de correções mas não no bloco de modelo do app `core` — ⚠️ inconsistência a resolver (deveria estar em `Usuario`).
- **CodigoDoisFatores** ✅
  - Finalidade: registrar códigos de 2FA de uso único.
  - Campos: `usuario` (FK), `codigo_hash` (`CharField(max_length=255)`), `criado_em`, `expira_em`, `usado` (bool), `tentativas` (inteiro).
  - Relacionamentos: `ForeignKey` para `Usuario` (`related_name="codigos_2fa"`).

## App `organizacao`
- **Departamento** ⚠️ — citado só pelo nome.
- **Cargo** ⚠️ — citado só pelo nome.

## App `funcionarios`
- **Funcionario** — parcialmente ✅: os campos base não são listados no plano (⚠️ lacuna), mas 3 campos novos são explicitamente especificados: `tipo_desligamento` (choices: `SEM_JUSTA_CAUSA`, `PEDIDO_DEMISSAO`, `JUSTA_CAUSA`, `ACORDO`), `aviso_previo_tipo` (choices: `INDENIZADO`, `TRABALHADO`, `DISPENSADO`), e `vaga_origem` (FK nullable para `recrutamento.Vaga`).
  - Relacionamentos conhecidos: FK para `Vaga` (`vaga_origem`); é referenciado por `FolhaPagamento.funcionario`.
- **Promocao** ⚠️ — citado só pelo nome.

## App `recrutamento`
- **Vaga** ⚠️ — citado só pelo nome; é o destino de `Funcionario.vaga_origem`.
- **Candidato** ⚠️ — citado só pelo nome.
- **Candidatura** — parcialmente ✅: campos não listados (⚠️), mas sabe-se que é referenciada por `ExtracaoCurriculo` via `OneToOneField`.

## App `ponto`
- **PontoMarcacao** ⚠️
- **PontoApuracaoDiaria** ⚠️ — citado como fonte de leitura de `gerar_lancamentos_ponto`, mas sem campos.
- **PontoCalendario** ⚠️
- **PontoOcorrencia** ⚠️

## App `ferias`
- **FeriasPeriodoAquisitivo** ⚠️ — citado como o que é criado automaticamente na contratação, sem campos.
- **FeriasSolicitacao** ⚠️
- **FeriasRegistro** ⚠️ — citado como insumo de `gerar_lancamentos_ferias`, sem campos.

## App `beneficios`
- **TipoBeneficio** ⚠️
- **FuncionarioBeneficio** ⚠️

## App `folha`
- **FolhaPagamento** ✅
  - Finalidade: representar uma folha (mensal ou de rescisão) de um funcionário em uma competência.
  - Campos: `funcionario` (FK), `competencia` (date), `status` (choices: `ABERTA`, `FECHADA`), `total_proventos`, `total_descontos`, `valor_liquido` (decimais), `fechada_em` (datetime, nullable), `fechada_por` (FK para `core.Usuario`, nullable). Ganha também um campo `tipo` com valor adicional `RESCISAO` (além do tipo mensal padrão, cujo nome exato não é dado — ⚠️ lacuna).
  - Relacionamentos: `related_name="lancamentos"` para `LancamentoFolha`; FK para `Funcionario`; FK para `Usuario` (quem fechou).
- **LancamentoFolha** ✅
  - Finalidade: cada linha de provento/desconto/informativo de uma folha.
  - Campos: `folha` (FK), `tipo` (choices: `PROVENTO`, `DESCONTO`, `INFORMATIVO`), `codigo` (ex.: `SALARIO`, `INSS`, `IRRF`, `FGTS`, `FERIAS`), `descricao`, `referencia` (base de cálculo, nullable), `valor`, `origem` (ex.: `CONTRATO`, `CALCULO_AUTOMATICO`, `FERIAS`, `BENEFICIO`).
  - Relacionamentos: FK para `FolhaPagamento`.
- **TabelaIRRF** ✅ — `vigente_desde`, `faixa_min`, `faixa_max` (nullable), `aliquota`, `parcela_deduzir`.
- **ParametroIRRF** ✅ — `ano` (unique), `valor_dependente`.
- **TabelaINSS** ✅ — `vigente_desde`, `faixa_min`, `faixa_max` (nullable), `aliquota`.

## App `solicitacoes`
- **Solicitacao** ⚠️
- **StatusSolicitacao** ⚠️
- **TipoSolicitacao** ⚠️ — nenhum dos três tem campo, fluxo ou regra descritos no plano.

## App `ia`
- **ExtracaoCurriculo** ✅
  - Finalidade: guardar o resultado bruto da extração de IA, separado dos dados confirmados.
  - Campos: `candidatura` (OneToOne para `recrutamento.Candidatura`), `dados_extraidos` (JSONField), `confirmado_pelo_rh` (bool), `criado_em`.
  - Relacionamentos: `OneToOneField` para `Candidatura`.

---

# 9. Contratos de API

⚠️ **Lacuna geral no plano**: nenhum endpoint tem payload (body) ou schema de resposta especificado — o plano dá apenas método HTTP, URL e uma observação textual curta. Os 7 endpoints abaixo são **todos** os endpoints citados no documento; nenhum outro app (`core`, `organizacao`, `funcionarios`, `recrutamento`, `ponto`, `ferias`, `beneficios`, `solicitacoes`, `relatorios`) tem endpoint algum especificado.

| Método | URL | Permissões (conforme plano) | Observação | Payload/Resposta |
|---|---|---|---|---|
| GET | `/api/folha/meus-holerites/` | ⚠️ não especificado (presumivelmente o próprio funcionário autenticado) | Substitui o mock do `payslips.js` — dados reais vindos de `LancamentoFolha` | ⚠️ Lacuna |
| GET | `/api/folha/<id>/` | ⚠️ não especificado | Inclui lista de lançamentos (proventos/descontos) | ⚠️ Lacuna |
| POST | `/api/folha/<id>/processar/` | "RH/DP" | Chama `processar_folha()`, nunca SQL direto | ⚠️ Lacuna |
| POST | `/api/folha/<id>/fechar/` | ⚠️ não especificado | Bloqueia novos lançamentos após fechada | ⚠️ Lacuna |
| POST | `/api/folha/<id>/reabrir/` | Só `ADMIN`/`DP_CHEFE` | — | ⚠️ Lacuna |
| GET | `/api/folha/<id>/pdf/` | ⚠️ não especificado | Gera o PDF dinamicamente a cada request | ⚠️ Lacuna |
| POST | `/api/folha/<id>/explicar/` | ⚠️ não especificado | Resposta cacheada (folha fechada não muda) | ⚠️ Lacuna |

**Endpoints mencionados sem método/URL definidos** (citados apenas como existência funcional, não como contrato de API):
- Upload de currículo (PDF/DOCX) — app `ia`.
- "Ver detalhes" / geração de PDF de perfil do candidato — app `ia`.
- Login e validação de 2FA — app `core` (descritos como fluxo, sem rota `/login` ou `/2fa` formalizada no plano novo).

---

# 10. Estrutura de Frontend

⚠️ **Lacuna geral no plano**: não há inventário de telas, wireframe ou lista de componentes. O plano define a stack de frontend ("Templates Django + JS vanilla, sem React por agora") mas só **2 telas são citadas nominalmente**:

| Tela | Objetivo (conforme plano) | Componentes / Ações citadas | APIs consumidas (citadas) |
|---|---|---|---|
| `meus-holerites` | Funcionário consulta os próprios holerites | Botão "Explicar com IA" | `GET /api/folha/meus-holerites/`, `GET /api/folha/<id>/`, `GET /api/folha/<id>/pdf/`, `POST /api/folha/<id>/explicar/` |
| Kanban de candidatos (referenciado como `recruitment.html`) | RH acompanha candidaturas e revisa dados extraídos por IA | Botão "Ver detalhes" (gera PDF de perfil + mostra dado bruto da IA) | ⚠️ não especificado |

**Todas as demais telas implícitas pela existência dos apps** (login, 2FA, dashboard, CRUD de funcionários/departamentos/cargos/vagas/candidatos, ponto, férias, benefícios, solicitações, relatórios) ⚠️ **não têm objetivo, componente, ação ou API consumida especificados no plano** — apenas a existência do app que as sustentaria.

---

# 11. Estratégia de Testes

## Testes unitários (services) — bem cobertos no plano
- `test_calculadora_irrf.py`: cada faixa da tabela progressiva, valores de borda, base zero, base negativa.
- `test_calculadora_inss.py`: idem, progressivo por faixa.
- `test_processador_folha.py`: fluxo completo abrir→processar→fechar; reprocessar folha fechada levanta `FolhaJaFechadaError`; reabrir permite reprocessar.
- `test_regras_por_tipo_desligamento.py`: confirma que justa causa não gera férias/13º proporcional nem aviso prévio indenizado; pedido de demissão não gera multa FGTS.
- `test_processador_rescisao.py`: cenário completo de demissão sem justa causa, ponta a ponta.
- Testes (sem nome de arquivo definido) para 2FA: gerar código e confirmar que o hash cabe no campo; código expirado, usado e errado; bloqueio após N tentativas.
- Testes (sem nome de arquivo definido) para cada calculadora de rescisão isolada (saldo, férias rescisórias, 13º proporcional, aviso prévio, multa FGTS).

## Testes de API
- `test_api_folha.py`: endpoint de geração de holerite retorna os mesmos valores calculados pelos services (sem duplicar lógica na view).
- `test_explicador_holerite.py` (app `ia`): mocka a API Claude, garante que não inventa valor fora do que foi passado.
- `test_extrator_curriculo.py` (app `ia`): mocka a API Claude, testa currículos de exemplo (texto limpo e malicioso).

## Testes de integração
⚠️ **Lacuna no plano**: não existe uma categoria de teste de integração distinta de unit/API no documento. A linha mais próxima é `test_processador_folha.py`/`test_processador_rescisao.py`, que testam a orquestração completa dos services entre si (mas via chamada direta de função Python, não via banco real + fila + e-mail, por exemplo).

## Testes E2E
⚠️ **Lacuna total no plano**: nenhuma ferramenta (Selenium, Playwright, Cypress) ou estratégia de teste E2E (ponta a ponta via navegador) é mencionada em nenhum ponto do documento.

---

# 12. Roadmap Técnico

Convertendo o cronograma do plano (fases 0–5 + margem) em Epic → Feature → Task. Tasks marcadas ⚠️ são inferidas apenas do nome do app, sem detalhe no plano.

## Epic 0 — Setup + 2FA (jun/2026, ~30h)
- **Feature: Projeto Django base**
  - Task: criar projeto, configurar `django-environ`, `.env`/`.env.example`, CI básico com `pytest`.
- **Feature: App `core` — auth e perfis**
  - Task: ⚠️ modelar `Usuario` customizado (campos não detalhados no plano).
  - Task: ⚠️ modelar permissões por perfil (perfis não enumerados no plano).
- **Feature: 2FA por e-mail**
  - Task: modelo `CodigoDoisFatores` (`codigo_hash` com `max_length=255`).
  - Task: service `gerar_codigo`/`validar_codigo` (`secrets.randbelow`, `make_password`/`check_password`, expiração, rate limiting).
  - Task: middleware de bloqueio de rota até 2FA verificado.
  - Task: testes de hash, expiração, uso único, bloqueio por tentativas.
- **Feature: App `organizacao`**
  - Task: ⚠️ modelar `Departamento`, `Cargo` (campos não detalhados no plano).

## Epic 1 — Domínio base (jul/2026, ~50h)
- **Feature: App `funcionarios`** — ⚠️ CRUD de `Funcionario`/`Promocao` via DRF + Admin (campos não detalhados).
- **Feature: App `ponto`** — ⚠️ CRUD de `PontoMarcacao`/`PontoApuracaoDiaria`/`PontoCalendario`/`PontoOcorrencia` (sem fluxo detalhado).
- **Feature: App `ferias`** — ⚠️ CRUD de `FeriasPeriodoAquisitivo`/`FeriasSolicitacao`/`FeriasRegistro` (sem fluxo de aprovação detalhado).
- **Feature: App `beneficios`** — ⚠️ CRUD de `TipoBeneficio`/`FuncionarioBeneficio` (sem detalhe nenhum).

## Epic 2 — Núcleo de folha (ago/2026, ~50h)
- **Feature: Modelos de folha** — `FolhaPagamento`, `LancamentoFolha`, `TabelaIRRF`, `ParametroIRRF`, `TabelaINSS`.
- **Feature: Calculadoras** — `calculadora_inss.py`, `calculadora_irrf.py`, `calculadora_fgts.py`, cada uma com teste unitário cobrindo faixas/bordas.
- **Feature: Integração ponto/férias → folha** — `gerar_lancamentos_ponto`, `gerar_lancamentos_ferias`, chamadas explícitas, com teste próprio.
- **Feature: Orquestração** — `processador_folha.py` (abrir → gerar lançamentos → calcular impostos → totalizar → fechar), `FolhaJaFechadaError`.
- **Feature: API e PDF** — os 6 endpoints de folha; geração de PDF dinâmica via `WeasyPrint`.

## Epic 2.5 — Rescisão (set/2026, ~40h)
- **Feature: Modelo** — campos `tipo_desligamento`, `aviso_previo_tipo` em `Funcionario`; `FolhaPagamento.tipo="RESCISAO"`.
- **Feature: Calculadoras de rescisão** — saldo de salário, férias rescisórias, 13º proporcional, aviso prévio, multa FGTS.
- **Feature: Regras por tipo de desligamento** — estrutura `REGRAS` testável.
- **Feature: Orquestração** — `processador_rescisao.py` (5 passos descritos na seção 7).

## Epic 3 — Recrutamento + Relatórios (out/2026, ~30h)
- **Feature: Correções do fluxo de contratação** — senha segura + `must_change_password`, signal de férias, `Funcionario.vaga_origem`.
- **Feature: App `relatorios`** — ⚠️ 2-3 relatórios (headcount por departamento, custo de folha por mês, férias vencendo); formato de saída não definido.

## Epic 4 — Docker + ajustes (nov/2026, ~25h)
- **Feature: `docker-compose`** — Django + Postgres.
- **Feature: Segurança** — ⚠️ "checklist OWASP básico" (itens não listados no plano).
- **Feature: Polimento de front** — templates + JS.

## Epic 5 — IA (dez/2026, ~40-48h)
- **Feature: Guia de holerites** — `client.py`, `explicador_holerite.py`, endpoint `explicar/`, cache, botão na UI.
- **Feature: Extração de currículo** — upload, `pdfplumber`/`python-docx`, sanitização, `extrator_curriculo.py`, modelo `ExtracaoCurriculo`, tela de revisão, PDF de perfil (`WeasyPrint`).

## Margem (jan–fev/2027)
- Buffer para imprevistos/polimento, ou avançar para Laravel (decisão em aberto, não comprometida no plano).
- README comparativo, seed/fixtures de demo e deploy — explicitamente fora deste documento, "entram quando você pedir".

---

# 13. Critérios de Aceite

Critérios derivados diretamente das regras e testes já descritos no plano. Funcionalidades sem detalhe suficiente estão marcadas ⚠️.

**Cálculo de IRRF**
- Dado uma base líquida e uma data de referência, quando existe faixa vigente (`vigente_desde <= data`) compatível com a base, então o valor calculado é `(base * aliquota) - parcela_deduzir`, nunca negativo (`max(..., 0)`).
- Dado que não existe faixa compatível, então o valor retornado é `0.00`.

**Cálculo de INSS / FGTS**
- ⚠️ O plano não descreve a fórmula de INSS/FGTS com o mesmo nível de detalhe do IRRF (só a assinatura da função `calcular_inss(base, ano)`/`calcular_fgts(base)`). Critério de aceite específico não pode ser derivado além de "progressivo por faixa" (INSS) e "valor único" (FGTS).

**Processamento de folha**
- Dado uma folha com `status="ABERTA"`, quando `processar_folha` é chamado, então lançamentos antigos de `origem="CALCULO_AUTOMATICO"` são removidos e recriados, e os totais são recalculados.
- Dado uma folha com `status="FECHADA"`, quando `processar_folha` é chamado, então o sistema levanta `FolhaJaFechadaError` e não altera nada.
- Dado uma folha fechada, quando um usuário que não é `ADMIN`/`DP_CHEFE` tenta reabri-la, então ⚠️ o plano não especifica o comportamento exato (presumivelmente 403, não confirmado).

**Rescisão**
- Dado um desligamento do tipo `JUSTA_CAUSA`, quando a rescisão é processada, então não são gerados lançamentos de férias proporcionais, 13º proporcional nem aviso prévio indenizado.
- Dado um desligamento do tipo `PEDIDO_DEMISSAO`, quando a rescisão é processada, então a multa de FGTS é `0%`.
- Dado um desligamento do tipo `SEM_JUSTA_CAUSA`, quando a rescisão é processada, então a multa de FGTS é `40%` sobre o saldo depositado.

**2FA**
- Dado um código gerado, quando armazenado, então o hash cabe inteiramente no campo `codigo_hash` (`max_length=255`) sem truncamento — este é o critério que corrige o bug do projeto original.
- Dado um código expirado ou já usado, quando informado pelo usuário, então a validação falha.
- Dado N tentativas erradas consecutivas, quando a tentativa N+1 ocorre, então o sistema bloqueia — ⚠️ valor de N não definido no plano.

**Extração de currículo por IA**
- Dado um currículo enviado, quando a IA extrai os dados, então o resultado bruto é salvo em `ExtracaoCurriculo` com `confirmado_pelo_rh=False`.
- Dado um dado extraído, quando o RH não confirma, então nenhum `Candidato`/`Candidatura` é criado.
- Dado um arquivo maior que 5MB ou de tipo real inválido (magic bytes), quando enviado, então é rejeitado antes de chegar à IA.

**Explicação de holerite por IA**
- Dado uma folha que não está fechada, quando a explicação é solicitada, então ⚠️ o plano não especifica se a operação é bloqueada ou permitida (a regra "recebe a folha já fechada" é descrita como premissa de design, não como validação explícita de erro).
- Dado uma folha fechada, quando a IA gera a explicação, então nenhum valor fora da lista de lançamentos recebida pode aparecer na resposta (restrição de prompt).

**Demais módulos (organizacao, funcionarios, recrutamento, ponto, ferias, beneficios, solicitacoes, relatorios)**
- ⚠️ **Lacuna no plano**: sem campos, fluxos ou regras detalhadas, não é possível derivar critérios de aceite específicos além do genérico "Definition of Done" (seção 14).

---

# 14. Checklist de Desenvolvimento

Baseado diretamente na seção "Definition of Done" do plano, expandido com os princípios repetidos ao longo do documento.

**Por feature, antes de considerar pronta:**
- [ ] Modelo + migration aplicada.
- [ ] Service com teste unitário (quando envolve cálculo/regra de negócio).
- [ ] Endpoint DRF testado (`test_api_*.py`).
- [ ] Tela/rota server-side renderizando dados reais — nunca mock, nunca placeholder.
- [ ] Rota adicionada ao `urls.py` **só neste momento**, nunca antes de existir o endpoint e a tela funcionando atrás.

**Princípios transversais (derivados de múltiplas seções do plano):**
- [ ] Nenhuma regra de negócio implementada como trigger/function de banco — só schema, constraints e índices.
- [ ] Nenhum segredo (chave de API, senha, JWT secret) hardcoded — tudo via `.env`/`django-environ`.
- [ ] Toda integração entre módulos (ex.: ponto/férias → folha) é uma chamada de função explícita ou signal documentado, nunca implícita.
- [ ] Toda criação de registro a partir de resposta de IA passa por confirmação humana antes de persistir.
- [ ] Todo upload de arquivo valida tipo real (magic bytes) e tamanho antes de processar.
- [ ] Todo campo de hash é dimensionado pelo tamanho do hash, não pelo dado original (lição do bug de 2FA).

---

# 15. Riscos Técnicos

Riscos herdados do projeto Java anterior, conforme documentados na seção "Contexto" e "Correções específicas" do plano.

| Risco herdado | Como o plano evita | Status no guia |
|---|---|---|
| Cálculo de holerite existir só no banco, nunca chamado pela aplicação | Toda regra de cálculo vive em services Python testáveis (`apps/folha/services/`) | Coberto (seção 6/7/12) |
| Tela de holerites 100% simulada (`Math.random()`/`localStorage`) | Endpoints reais consumindo `LancamentoFolha`; PDF gerado dos dados reais | Coberto |
| Rota quebrada apontando para view inexistente | "Definition of Done": rota só entra no `urls.py` com tela funcionando atrás | Coberto (regra de processo) |
| Configuração de segurança referenciando módulos nunca implementados | Mesma regra de processo acima, aplicada a toda nova rota | Coberto (regra de processo) |
| Senha de contratação previsível (CPF + sufixo fixo) | Senha aleatória (`secrets.token_urlsafe`) + troca obrigatória no primeiro login | Coberto |
| Segredos reais commitados no git | `django-environ`, `.env` fora do git, `.env.example` commitado | Coberto |
| Desligamento sem nenhum cálculo de rescisão financeira | App `folha`/rescisão completo (saldo, férias proporcionais, 13º, aviso prévio, multa FGTS) | Coberto (seção 6/7/12) |
| Bug de schema no 2FA (hash truncado em campo curto) | `CodigoDoisFatores.codigo_hash` dimensionado para o hash desde o início | Coberto |
| Ausência de rastreabilidade vaga → funcionário contratado | `Funcionario.vaga_origem` (FK nullable) | Coberto |
| Ponto e férias desconectados da folha | `gerar_lancamentos_ponto`/`gerar_lancamentos_ferias` chamados explicitamente por `processador_folha` | Coberto |

## ⚠️ Riscos não cobertos pelo plano (identificados neste guia, sem mitigação definida)

- **Ausência de auditoria/log de alterações**: o projeto original tinha um trigger de auditoria LGPD; o plano novo não propõe nenhum substituto.
- **Ausência de requisitos de performance, disponibilidade e escalabilidade**: nenhum SLA, número de usuários esperado ou estratégia de carga é definido.
- **Falta de contrato de API completo**: sem payload/resposta especificados, há risco de retrabalho entre back e front durante a implementação.
- **Falta de inventário de telas**: sem lista de páginas, há risco de subestimar o esforço de frontend (só 2 telas são nominalmente conhecidas).
- **Ausência de testes E2E**: nenhuma camada de teste valida o sistema integrado via navegador, só unidade e API.
- **Apps quase sem especificação** (`organizacao`, `funcionarios` CRUD manual, `recrutamento` ponta a ponta, `ponto`, `ferias` solicitação/aprovação, `beneficios`, `solicitacoes`): risco de subestimar horas do cronograma, já que essas fases (0, 1, 3) têm muito menos detalhe de design do que `folha`/`ia`/rescisão/2FA, apesar de horas alocadas.

