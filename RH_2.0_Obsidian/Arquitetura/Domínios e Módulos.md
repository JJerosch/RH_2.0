# Domínios e Módulos

Laravel não tem "apps" isolados como o Django. A separação por domínio é feita com **pastas manuais** em `app/Domain/` (decisão D-05 em [[Registro de Decisões]]).

```
app/
├── Domain/
│   ├── Core/          # Usuario, autenticação, perfis, 2FA → [[Autenticação e 2FA]]
│   ├── Organizacao/   # Departamento, Cargo
│   ├── Funcionarios/  # Funcionario, Promocao, Dependente
│   ├── Recrutamento/  # Vaga, Candidato, Candidatura
│   ├── Ponto/         # PontoMarcacao, PontoApuracaoDiaria, Escala, Local(filial)
│   ├── Ferias/        # FeriasPeriodoAquisitivo, FeriasSolicitacao, FeriasRegistro
│   ├── Beneficios/    # TipoBeneficio, FuncionarioBeneficio
│   ├── Folha/         # 🔑 núcleo → [[Folha de Pagamento]]
│   ├── Solicitacoes/  # Solicitacao (fluxo de aprovação + log de negócio)
│   ├── Relatorios/    # relatórios (nunca existiu no original)
│   └── Ia/            # 🤖 diferencial → [[Inteligência Artificial]]
├── Filament/          # Resources do painel admin (RH/DP)
└── Http/Controllers/Api/  # controllers REST (só Folha, por D-14)
```

Cada domínio tem tipicamente `Models/` e `Services/`. A regra: **model só guarda dados; service tem a lógica** (D-02).

## Destaques de cada módulo

### Funcionarios
Campos aprovados como MVP (D-17): pessoais (nome, nascimento, sexo, estado civil, nome da mãe), documentos (**CPF único, PIS/PASEP**, RG, CTPS...), contato/endereço, vínculo (matrícula, cargo, departamento, gestor, admissão, tipo de contrato, regime, status, filial), remuneração (salário base), bancários.
- **Dependente** = tabela à parte (cada dependente reduz a base do IRRF).
- **Promocao** = histórico salarial/cargo, nunca sobrescreve o "de/para". Reforçado por activitylog (D-13).

### Ponto (Q5 fechado — D-23/24/25)
"Agenda" com, por dia, quantos bateram ponto e quantos faltam. Batida liberada só em **localização válida** (multi-filial); `PontoMarcacao` registra funcionário, dia, hora e lugar. Ignora `PontoOcorrencia`/`PontoCalendario` do modelo antigo.

**Da marcação à apuração (D-23):**
- Batidas do dia são **ordenadas por horário e pareadas** (entrada↔saída, 2ª entrada↔2ª saída…); soma dos intervalos = horas trabalhadas.
- Um **job diário (scheduler)** fecha a `PontoApuracaoDiaria` do dia anterior.

**Tolerância (D-24):** padrão **CLT de 10 min/dia**, **configurável por empresa/tenant**. Dentro da tolerância não gera atraso nem hora extra.

**Jornada esperada (D-25):** entidade **`Escala`** (dias + horários, ex.: seg–sex 08:00–12:00 / 13:00–17:48), **atribuída ao funcionário**. Suporta turnos diferentes e multi-filial. A apuração compara trabalhado × escala → gera atraso/falta ou hora extra, que viram `LancamentoFolha` via `GeradorLancamentos` (ver [[Folha de Pagamento]]).

### Ferias (Q6 fechado — D-26/27/28)
**Fluxo (D-26):** funcionário solicita pelo portal Blade → **gestor** do time aprova (via contexto, D-20) → gera `Solicitacao` ao **Chefe de RH**, que dá o ok final. Duas aprovações.

**Período aquisitivo (D-27):** padrão **CLT** (12 meses trabalhados = 30 dias; período concessivo de 12 meses; "férias em dobro" se vencer), **configurável por tenant**. Um **job** monitora vencimento, alerta e abre novo período aquisitivo. `FeriasPeriodoAquisitivo` guarda o saldo.

**Escopo MVP (D-28 — CLT completa):**
- **Fracionamento** em até 3 períodos (um deles ≥14 dias) — com validação.
- **Abono pecuniário**: vender 1/3 (10 dias) — vira verba na folha.
- Recibo de férias = `FolhaPagamento` tipo `FERIAS` (D-16) com **1/3 constitucional**; abono entra como lançamento próprio. Ver [[Folha de Pagamento]].

### Recrutamento (Q7 fechado — D-29/30/31)
Gestor solicita vaga → Chefe de RH aprova → só Chefe de RH + gestor da vaga mexem no Kanban → IA extrai dados do currículo (revisão humana) → DP completa a "prévia de funcionário" → efetiva o `Funcionario`. CPF repetido mostra aviso referenciando o funcionário existente (definir LGPD depois).

**Kanban (D-29):** `Inscrito → Triagem → Entrevista RH → Entrevista Gestor → Proposta → Contratado`, com raia lateral **Reprovado** acessível de qualquer etapa.

**Gatilho da prévia (D-30):** ao entrar em **"Proposta"**, cria-se a **prévia de funcionário**; o DP preenche os dados que faltam; mover pra **"Contratado"** efetiva o `Funcionario` de verdade.

**Extração por IA (D-31):** ver formato padrão em [[Inteligência Artificial]]. A IA só sugere; RH revisa antes de virar candidato (D-12).

### Solicitacoes
Fluxo de aprovação **e** log das ações importantes: uma contratação passa por uma cadeia de solicitações; consultar as aprovadas mostra a trilha. Assistente de RH → chefe de RH; assistente de DP → chefe de DP. Ver [[Perfis de Acesso]].

### Relatorios
D-18: página Filament com filtros + CSV + PDF nos formais. Lista fechada em 6 (3 no MVP).
