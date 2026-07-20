# Perfis de Acesso

Definição de quem pode o quê. Pergunta Q1 **fechada** (ver decisões D-20/D-21 em [[Registro de Decisões]]).

## Perfis (lista fechada de roles)

| Perfil | Papel |
|---|---|
| **ADMIN** | administração do sistema |
| **Chefe de RH** | aprova contratações, controla o Kanban da vaga, eventos, cultura |
| **Chefe de DP** | controla benefícios, salário, folha |
| **Assistente de RH** | manda solicitações → Chefe de RH |
| **Assistente de DP** | manda solicitações → Chefe de DP |
| **Funcionário** | (de qualquer área) usa o portal: holerite, ponto, férias, solicitações |

## Modelo de permissão: perfil base + contexto (D-20)

Permissão = **role fixa (RBAC)** + **vínculos contextuais**, combinados via **Policies do Laravel**.

- **"Gestor" NÃO é um 7º perfil** (D-21) — é um **atributo**: um `Funcionario` marcado como responsável por um **departamento** e/ou uma **vaga**. Ele continua sendo funcionário (tem holerite, bate ponto) e ganha poder extra só no contexto do que lidera.
- Exemplos que o modelo precisa cobrir:
  - "Chefe de RH aprova tudo de RH" → via **role**.
  - "Gestor aprova férias só do próprio time" → via **contexto** (gestor do departamento daquele funcionário).
  - "Só o gestor da vaga X mexe no Kanban X" → via **contexto** (gestor daquela vaga).
- Implementação: cada ação sensível tem uma **Policy** que checa role E vínculo. Liga com [[APIs REST e Sanctum]] (403 quando falha).

## Divisão RH × DP (regra prática)

- **DP** cuida de **dinheiro e vínculo**: salário, benefícios, folha, dados do funcionário.
- **RH** cuida de **pessoas e processos**: contratações, eventos (ex.: festa da empresa), cultura.

## Fluxo de solicitações (o "roteamento")

- Assistente de RH manda solicitação → **Chefe de RH** aprova.
- Assistente de DP manda solicitação → **Chefe de DP** aprova.
- Gestores mandam solicitações (ex.: abrir vaga) → **Chefe de RH** aprova.
- Solicitações de **férias**: funcionário → aprovada pelo gestor dele → gera solicitação ao Chefe de RH.

Tudo isso passa pelo módulo `Solicitacoes`, que também serve de **log** (ver [[Domínios e Módulos]]).

## Recrutamento — quem mexe no quê
- Gestor solicita abertura de vaga; **Chefe de RH** aprova.
- Com a vaga criada, só **Chefe de RH + gestor da vaga** mexem no Kanban.
- **DP** define os dados do novo funcionário na "prévia de funcionário" quando ele é contratado.

## Cadeia de aprovação de contratação (D-22 — confirmada)

1. **Gestor** solicita abertura de vaga.
2. **Chefe de RH** aprova a abertura.
3. **Chefe de RH + gestor da vaga** tocam o Kanban de candidatos.
4. Candidato aprovado vira **"prévia de funcionário"**.
5. **DP** preenche os dados e **efetiva** o `Funcionario`.

Cada passo gera uma `Solicitacao` (serve de log — ver [[Domínios e Módulos]]). Detalhes do Kanban e da "prévia" seguem em aberto no **Q7** ([[Status das Perguntas]]).

> [!note] Ainda parkeado neste tema
> Só o **Q7** (etapas exatas do Kanban, gatilho da "prévia", formato da extração por IA). O modelo de perfis/permissão em si está fechado.
