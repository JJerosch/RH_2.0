# RHParaTodos-Django — Plano Técnico de Reconstrução

## Contexto

O RHParaTodos original (Java/Spring Boot + Thymeleaf + PostgreSQL) ficou incompleto e com funcionalidades confusas. Auditoria do projeto em `C:\Users\joaoj\Documents\GitHub\RHParaTodos` encontrou:

1. **Cálculo de holerite (INSS/IRRF/FGTS) existe só no banco**, em ~15 functions/triggers PL/pgSQL (`V1__baseline.sql`), e nunca é chamado pela aplicação Java — zero entidade/service/controller toca essas tabelas.
2. **Tela de holerites é 100% fake**: `payslips.js` e `employee.js` geram dados com `Math.random()` em `localStorage`, sem nenhuma chamada de API. Há duas implementações mock divergentes da mesma feature.
3. **Rota `/payslips` quebrada**: o controller mapeia para uma view que não existe nos templates (erro 500 se acessada).
4. **`SecurityConfig` referencia módulos nunca implementados**: `/payroll/**`, `/reports/**`, `/performance/**`, `/training/**` têm regra de autorização mas nenhum controller — projeto "esqueletado" maior do que o entregue.
5. **Contratação automática gera senha previsível** (6 dígitos do CPF + sufixo fixo) e não inicializa férias do novo funcionário.
6. **Segredos reais commitados no git** (senha de app do Gmail, JWT secret) em `application.properties`, rastreados desde commits antigos.

Este documento define como reconstruir o sistema em **Django + DRF**, evitando repetir esses erros — em especial, **toda regra de negócio de folha de pagamento vive em código Python testável, nunca em função/trigger de banco**.

**Janela de tempo**: 7-8 meses (jun 2026 – jan/fev 2027), sem prazo rígido — 6 meses era só o teto de viabilidade inicial. Dentro do plano PACER já existente (`plano-fullstack.html`, Fase 1 + Fase 2). Stack travada: **Django + DRF + templates Django/JS vanilla, sem React por agora**.

**Diferencial de IA**: o projeto inclui duas features de inteligência artificial (Claude API) como diferencial de portfólio — guia de holerites em linguagem natural e extração automática de dados de currículo. Ver seção dedicada "App `ia`".

---

## Decisão de arquitetura: por que nada de trigger/function no banco

O banco (PostgreSQL) fica restrito a:
- Schema (tabelas, colunas, tipos)
- Constraints de integridade: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK` simples (ex.: `valor >= 0`)
- Índices

Tudo que é **regra de negócio** — cálculo de INSS/IRRF progressivo, geração de lançamentos de folha, fechamento/reabertura de folha, elegibilidade de férias, etc. — vive em **services Python puros**, sem depender de Django ORM dentro da lógica de cálculo (facilita testar com `pytest` sem precisar de banco). Motivo prático: tabela de IRRF/INSS muda todo ano por lei — isso precisa estar em código revisável por PR e coberto por teste unitário, não escondido em SQL que ninguém revisa.

Justificativa adicional, ligada ao erro #4 do projeto anterior: **nenhuma rota entra no `urls.py` sem o endpoint e a tela atrás dela já funcionando**. Isso é regra de processo, não só de código — ver seção "Definition of Done" no final.

---

## Stack

- **Backend**: Django 5.x + Django REST Framework
- **Banco**: PostgreSQL (mesmo motor do projeto original, facilita comparar)
- **Auth**: `djangorestframework-simplejwt` (JWT) + sessão Django para as views server-rendered
- **Front**: Templates Django (Django Template Language) + JS vanilla (`fetch`) para partes dinâmicas — mesmo padrão que já existia no Spring (Thymeleaf + JS vanilla), reaproveita o que você já sabe
- **Config/segredos**: `django-environ`, `.env` fora do git, `.env.example` commitado
- **Testes**: `pytest-django`, focado nos services de folha e nos endpoints críticos
- **Migrations**: Django migrations nativas (substituem o Flyway)
- **IA**: SDK oficial `anthropic` (Claude), `pdfplumber` + `python-docx` para extração de texto de currículo
- **PDF**: `WeasyPrint` (renderiza template HTML → PDF) — perfil do candidato e holerite

---

## Estrutura de apps Django

```
rhparatodos/
├── config/                  # settings, urls raiz, wsgi/asgi
├── apps/
│   ├── core/                  # Usuario customizado, autenticação, permissões por perfil, 2FA por e-mail (ver seção dedicada)
│   ├── organizacao/           # Departamento, Cargo
│   ├── funcionarios/          # Funcionario, Promocao
│   ├── recrutamento/          # Vaga, Candidato, Candidatura
│   ├── ponto/                  # PontoMarcacao, PontoApuracaoDiaria, PontoCalendario, PontoOcorrencia
│   ├── ferias/                 # FeriasPeriodoAquisitivo, FeriasSolicitacao, FeriasRegistro
│   ├── beneficios/             # TipoBeneficio, FuncionarioBeneficio
│   ├── folha/                  # 🔑 núcleo da reconstrução — ver seção dedicada abaixo
│   ├── solicitacoes/           # Solicitacao, StatusSolicitacao, TipoSolicitacao
│   ├── relatorios/             # módulo que NUNCA existiu no projeto original — entra desde o início
│   └── ia/                     # 🤖 diferencial — guia de holerites + extração de currículo (ver seção dedicada)
└── tests/
```

Cada app = mesma divisão de domínio do projeto Spring (`Funcionario`, `Vaga`, `Candidatura`, etc.), só que sem o vácuo entre "tabela existe" e "regra de negócio existe".

---

## App `core` — autenticação e 2FA por e-mail

No projeto original, o 2FA por e-mail estava **bem desenhado, mas com um bug de schema fatal**: `TwoFactorCodeStore.java` inseria o hash bcrypt do código (~60 caracteres) numa coluna `varchar(10)` — truncava e quebrava em runtime sempre que um código era gerado. Fora esse bug, o fluxo (gerar código → enviar e-mail → bloquear rotas até validar) estava certo. A reconstrução mantém a mesma arquitetura, só corrigindo a causa raiz: nunca dimensionar um campo de hash pelo tamanho do dado original.

### Modelo

```python
# apps/core/models.py
class CodigoDoisFatores(models.Model):
    usuario = models.ForeignKey(Usuario, on_delete=models.CASCADE, related_name="codigos_2fa")
    codigo_hash = models.CharField(max_length=255)   # hash, nunca o código puro — campo dimensionado pro hash, não pro código de 6 dígitos
    criado_em = models.DateTimeField(auto_now_add=True)
    expira_em = models.DateTimeField()
    usado = models.BooleanField(default=False)
    tentativas = models.PositiveSmallIntegerField(default=0)
```

### Service (`apps/core/services/dois_fatores.py`)

- `gerar_codigo(usuario)`: gera 6 dígitos com `secrets.randbelow` (não `random`, que não é criptograficamente seguro), guarda o hash via `django.contrib.auth.hashers.make_password` (reaproveita a infra de hashing do Django, não reinventa), define expiração curta (ex.: 5 min), invalida códigos anteriores não usados do mesmo usuário, envia e-mail via `django.core.mail.send_mail`.
- `validar_codigo(usuario, codigo_informado)`: busca o código válido mais recente (não expirado, não usado), compara com `check_password`, incrementa `tentativas` e bloqueia depois de N tentativas erradas (rate limiting básico — melhoria em relação ao original, que não tinha limite de tentativas), marca como usado se válido.
- Teste dedicado que o projeto original nunca teve: gerar um código e confirmar que o hash cabe no campo, testar código expirado, usado e errado, e testar o bloqueio após N tentativas.

### Middleware (equivalente ao `TwoFactorEnforcementFilter`)

Middleware customizado em `apps/core/middleware.py` que verifica uma flag de sessão (`request.session["2fa_verificado"]`) e redireciona para `/2fa` se ausente, exceto em rotas isentas (`/login`, `/2fa`, estáticos) — mesma ideia do filtro Spring, só que como middleware Django.

---

## App `folha` — núcleo da reconstrução

### Modelos (apenas dados, sem lógica)

```python
class FolhaPagamento(models.Model):
    funcionario = models.ForeignKey(...)
    competencia = models.DateField()  # primeiro dia do mês
    status = models.CharField(choices=[("ABERTA", ...), ("FECHADA", ...)])
    total_proventos = models.DecimalField(...)
    total_descontos = models.DecimalField(...)
    valor_liquido = models.DecimalField(...)
    fechada_em = models.DateTimeField(null=True)
    fechada_por = models.ForeignKey("core.Usuario", null=True, ...)

class LancamentoFolha(models.Model):
    folha = models.ForeignKey(FolhaPagamento, related_name="lancamentos", ...)
    tipo = models.CharField(choices=[("PROVENTO", ...), ("DESCONTO", ...), ("INFORMATIVO", ...)])
    codigo = models.CharField(max_length=20)   # SALARIO, INSS, IRRF, FGTS, FERIAS...
    descricao = models.CharField(max_length=120)
    referencia = models.DecimalField(null=True)  # base de cálculo usada
    valor = models.DecimalField(...)
    origem = models.CharField(...)  # CONTRATO, CALCULO_AUTOMATICO, FERIAS, BENEFICIO

class TabelaIRRF(models.Model):
    vigente_desde = models.DateField()
    faixa_min = models.DecimalField(...)
    faixa_max = models.DecimalField(null=True)
    aliquota = models.DecimalField(...)
    parcela_deduzir = models.DecimalField(...)

class ParametroIRRF(models.Model):
    ano = models.IntegerField(unique=True)
    valor_dependente = models.DecimalField(...)

class TabelaINSS(models.Model):
    vigente_desde = models.DateField()
    faixa_min = models.DecimalField(...)
    faixa_max = models.DecimalField(null=True)
    aliquota = models.DecimalField(...)
```

Sem `Meta.triggers`, sem `RunSQL` de function. Tudo que era `CREATE FUNCTION` no projeto anterior se torna uma das classes abaixo.

### Services (regra de negócio, 100% Python, 100% testável)

```
apps/folha/services/
├── calculadora_inss.py     # calcular_inss(base, ano) -> Decimal
├── calculadora_irrf.py     # calcular_irrf(base_liquida, data_referencia) -> Decimal
├── calculadora_fgts.py     # calcular_fgts(base) -> Decimal
├── gerador_lancamentos.py  # monta os LancamentoFolha (salário, benefícios, horas extra/falta vindas do ponto, férias)
└── processador_folha.py    # orquestra: abrir folha -> gerar lançamentos -> calcular impostos -> totalizar -> fechar
```

**Achado da auditoria que motiva o design abaixo**: no projeto original, `FeriasService` e `PontoService` nunca chamavam as functions de folha (`gerar_lancamentos_ferias`, `gerar_eventos_ponto_folha`) — não é só a folha que estava isolada, é **todo módulo que deveria alimentar a folha que estava desconectado dela**. Pra não repetir isso, a integração é explícita e tem dono:

```python
# apps/folha/services/gerador_lancamentos.py
def gerar_lancamentos_ponto(folha: "FolhaPagamento", competencia: date) -> None:
    """Lê PontoApuracaoDiaria do período e cria lançamentos HEXTRA/FALTA. Chamado explicitamente por processar_folha."""

def gerar_lancamentos_ferias(folha: "FolhaPagamento", ferias_registro: "FeriasRegistro") -> None:
    """Cria lançamentos FERIAS/FERIAS_1_3 quando uma FeriasRegistro é aprovada pra competência da folha aberta."""
```

Cada uma chamada de forma explícita (por `processador_folha` ou por um Django signal documentado em comentário, nunca implícito) e com teste próprio — exatamente o ponto onde o projeto original quebrou silenciosamente.

Exemplo de contrato (espelha 1:1 o que a function `calcular_irrf_progressivo` fazia no SQL, só que testável):

```python
# apps/folha/services/calculadora_irrf.py
from decimal import Decimal, ROUND_HALF_UP
from datetime import date
from apps.folha.models import TabelaIRRF

def calcular_irrf(base_liquida: Decimal, data_referencia: date) -> Decimal:
    faixa = (
        TabelaIRRF.objects
        .filter(vigente_desde__lte=data_referencia)
        .filter(models.Q(faixa_max__gte=base_liquida) | models.Q(faixa_max__isnull=True))
        .filter(faixa_min__lte=base_liquida)
        .order_by("-vigente_desde")
        .first()
    )
    if faixa is None:
        return Decimal("0.00")
    valor = (base_liquida * faixa.aliquota) - faixa.parcela_deduzir
    return max(valor.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP), Decimal("0.00"))
```

```python
# apps/folha/services/processador_folha.py
from django.db import transaction
from .calculadora_inss import calcular_inss
from .calculadora_irrf import calcular_irrf
from .calculadora_fgts import calcular_fgts

class FolhaJaFechadaError(Exception): ...

@transaction.atomic
def processar_folha(folha_id: int) -> "FolhaPagamento":
    folha = FolhaPagamento.objects.select_for_update().get(pk=folha_id)
    if folha.status == "FECHADA":
        raise FolhaJaFechadaError(f"Folha {folha_id} está fechada.")

    folha.lancamentos.filter(origem="CALCULO_AUTOMATICO").delete()

    base_inss = _base_inss(folha)
    valor_inss = calcular_inss(base_inss, folha.competencia.year)
    _registrar_lancamento(folha, "DESCONTO", "INSS", base_inss, valor_inss)

    base_irrf = _base_irrf(folha) - valor_inss
    valor_irrf = calcular_irrf(base_irrf, folha.competencia)
    _registrar_lancamento(folha, "DESCONTO", "IRRF", base_irrf, valor_irrf)

    valor_fgts = calcular_fgts(_base_fgts(folha))
    _registrar_lancamento(folha, "INFORMATIVO", "FGTS", _base_fgts(folha), valor_fgts)

    _recalcular_totais(folha)
    return folha
```

Cada função (`calcular_inss`, `calcular_irrf`, `calcular_fgts`, `processar_folha`) ganha teste unitário com tabelas de exemplo — isso é o que o projeto anterior nunca teve e que o banco não permitia ter de forma simples.

### Testes mínimos obrigatórios (`apps/folha/tests/`)

- `test_calculadora_irrf.py`: cada faixa da tabela progressiva, valores de borda (exatamente no limite de faixa), base zero, base negativa (deve recusar ou zerar).
- `test_calculadora_inss.py`: idem, progressivo por faixa.
- `test_processador_folha.py`: fluxo completo abrir → processar → fechar; tentar reprocessar folha fechada deve levantar `FolhaJaFechadaError`; reabrir folha permite reprocessar.
- `test_api_folha.py`: endpoint de geração de holerite retorna os mesmos valores calculados pelos services (sem duplicar lógica na view).

---

## API (DRF) — só entra rota com tela funcionando atrás

Mesma exigência que faltou no projeto original (`/payroll`, `/reports` existiam no Security mas não no resto):

| Recurso | Endpoint | Observação |
|---|---|---|
| Folha do funcionário | `GET /api/folha/meus-holerites/` | Substitui o mock do `payslips.js` — dados reais vindos de `LancamentoFolha` |
| Detalhe de um holerite | `GET /api/folha/<id>/` | Inclui lista de lançamentos (proventos/descontos) |
| Processar folha (RH/DP) | `POST /api/folha/<id>/processar/` | Chama `processar_folha()`, nunca SQL direto |
| Fechar folha | `POST /api/folha/<id>/fechar/` | Bloqueia novos lançamentos após fechada (checagem em Python, não trigger) |
| Reabrir folha | `POST /api/folha/<id>/reabrir/` | Só ADMIN/DP_CHEFE |
| PDF do holerite | `GET /api/folha/<id>/pdf/` | Gera o PDF **dinamicamente a cada request** — ver decisão abaixo |

### Geração do PDF do holerite: dinâmico, sem armazenar arquivo

Decisão: o PDF **não é gerado uma vez e guardado** (nem localmente, nem em nuvem) — é renderizado on-demand a cada download, direto dos `LancamentoFolha` daquela `FolhaPagamento` via `WeasyPrint`.

- A fonte de verdade é imutável depois que a folha está `FECHADA` (`processador_folha` já bloqueia reprocessamento — ver `FolhaJaFechadaError`), então renderizar na hora sempre reproduz o mesmo conteúdo financeiro. Não existe risco de "perder" o holerite histórico.
- Evita depender de storage em nuvem (bucket, lifecycle, custo) sem necessidade real nesse estágio do projeto — isso é algo que folhas de pagamento em grande escala adicionam por motivo de prova jurídica (assinatura digital, hash), não por necessidade técnica de exibir o documento.
- Mais simples de testar: só precisa validar a renderização do template com dados fixos, sem mockar storage externo.
- Ressalva conhecida: se o layout do template mudar no futuro, um holerite antigo reimpresso herda a aparência nova — os valores continuam corretos (o que importa legalmente), só a formatação visual não é "congelada". Se o projeto um dia virar produção real com obrigação trabalhista, vale revisitar e passar a congelar o arquivo no momento do fechamento da folha — não necessário agora.

---

## App `folha` — rescisão (desligamento com cálculo financeiro completo)

**Achado da auditoria**: no projeto original, desligar um funcionário (`SolicitacaoService.executarDesativacaoFuncionario`) só muda `status` para `INATIVO` e grava `dataDesligamento` — zero cálculo de saldo de salário, férias proporcionais, 13º proporcional, aviso prévio ou multa de FGTS, e nenhum lançamento na folha. Isso é tratado como mudança de status administrativo, não como evento financeiro. A reconstrução trata rescisão como um processamento de folha de verdade, reaproveitando as calculadoras de INSS/IRRF já existentes.

### Modelo (campos novos em `Funcionario`)

```python
class Funcionario(models.Model):
    ...
    tipo_desligamento = models.CharField(null=True, choices=[
        ("SEM_JUSTA_CAUSA", ...), ("PEDIDO_DEMISSAO", ...),
        ("JUSTA_CAUSA", ...), ("ACORDO", ...),   # acordo = CLT art. 484-A
    ])
    aviso_previo_tipo = models.CharField(null=True, choices=[
        ("INDENIZADO", ...), ("TRABALHADO", ...), ("DISPENSADO", ...),
    ])
```

`FolhaPagamento.tipo` ganha um valor `RESCISAO` (além do mensal padrão) — reaproveita a mesma estrutura de `LancamentoFolha`, PDF e explicação por IA já desenhada pra folha mensal.

### Services (`apps/folha/services/rescisao/`)

```
apps/folha/services/rescisao/
├── calculadora_saldo_salario.py            # dias trabalhados no mês / 30 * salário
├── calculadora_ferias_rescisao.py          # férias vencidas (se houver) + proporcionais, ambas com 1/3
├── calculadora_decimo_terceiro_proporcional.py
├── calculadora_aviso_previo.py             # indenizado vs trabalhado vs dispensado
├── calculadora_multa_fgts.py               # 40% (sem justa causa) ou 20% (acordo) sobre o saldo de FGTS depositado
├── regras_por_tipo_desligamento.py         # tabela explícita: o que cada tipo de desligamento perde/mantém direito
└── processador_rescisao.py                 # orquestra tudo, gera FolhaPagamento tipo RESCISAO
```

`regras_por_tipo_desligamento.py` é o ponto mais sensível — cada tipo de desligamento muda quais verbas se aplicam (ex.: justa causa não tem direito a férias proporcionais, 13º proporcional nem aviso prévio indenizado; pedido de demissão não gera multa de FGTS). Fica como uma estrutura de dados explícita e testável, não um `if/else` espalhado:

```python
REGRAS = {
    "SEM_JUSTA_CAUSA": {"ferias_proporcionais": True, "decimo_proporcional": True, "aviso_previo_indenizado": True, "multa_fgts_pct": Decimal("40")},
    "PEDIDO_DEMISSAO": {"ferias_proporcionais": True, "decimo_proporcional": True, "aviso_previo_indenizado": False, "multa_fgts_pct": Decimal("0")},
    "JUSTA_CAUSA":     {"ferias_proporcionais": False, "decimo_proporcional": False, "aviso_previo_indenizado": False, "multa_fgts_pct": Decimal("0")},
    "ACORDO":          {"ferias_proporcionais": True, "decimo_proporcional": True, "aviso_previo_indenizado": True, "multa_fgts_pct": Decimal("20")},
}
```

`processador_rescisao(funcionario_id, tipo_desligamento, data_desligamento, aviso_previo_tipo)`:
1. Consulta `REGRAS[tipo_desligamento]` pra saber quais verbas calcular.
2. Calcula saldo de salário, férias (vencidas + proporcionais), 13º proporcional, aviso prévio — só os que a regra permite.
3. Aplica `calcular_inss`/`calcular_irrf` (reaproveitados da folha mensal) sobre saldo de salário e 13º separadamente — são bases de cálculo distintas.
4. Calcula multa de FGTS sobre o histórico de FGTS depositado (soma dos lançamentos `FGTS` em `LancamentoFolha` ao longo do vínculo).
5. Gera `FolhaPagamento` tipo `RESCISAO` com todos os lançamentos, fecha automaticamente (evento único, sem necessidade de etapa manual de fechamento).

### Testes obrigatórios

- Cada calculadora isolada (saldo, férias rescisórias, 13º proporcional, aviso prévio, multa FGTS).
- `test_regras_por_tipo_desligamento.py`: confirma que justa causa não gera férias/13º proporcional nem aviso prévio indenizado; que pedido de demissão não gera multa FGTS; etc.
- `test_processador_rescisao.py`: cenário completo de demissão sem justa causa, ponta a ponta.

---

## App `ia` — diferenciais de inteligência artificial

Centraliza tudo que chama a API da Anthropic (Claude), pra não espalhar chave/cliente HTTP pelo projeto.

```
apps/ia/
├── client.py                      # wrapper fino sobre o SDK anthropic (model, timeout, retries)
├── services/
│   ├── explicador_holerite.py     # explica em português um holerite JÁ calculado
│   └── extrator_curriculo.py      # extrai texto de PDF/DOCX e pede JSON estruturado à IA
└── tests/
    ├── test_explicador_holerite.py   # mocka a API, garante que não inventa valor fora do que foi passado
    └── test_extrator_curriculo.py    # mocka a API, testa currículos de exemplo (texto limpo e malicioso)
```

### 1. Guia de holerites

- Recebe a `FolhaPagamento` **já fechada** + seus `LancamentoFolha` — a IA nunca calcula nada, só explica números que `processador_folha` já produziu. Isso elimina o risco de a IA "inventar" um valor de imposto.
- Prompt restrito: passa os lançamentos como dado estruturado e pede explicação em português simples, com instrução explícita de não declarar nenhum valor que não esteja na lista recebida.
- Endpoint: `POST /api/folha/<id>/explicar/`. Resposta cacheada (`ExplicacaoHolerite` ou cache de Django) — folha fechada não muda, não precisa gerar de novo a cada clique.
- Botão "Explicar com IA" na tela de holerite do funcionário (`meus-holerites`).

### 2. Extração de dados de currículo

- Upload de PDF/DOCX na tela pública/RH de candidatura (reaproveita o campo `Candidato.curriculo_url`, mas agora o arquivo é processado, não só guardado).
- `pdfplumber` (PDF) / `python-docx` (DOCX) extraem o texto bruto.
- Texto **sanitizado antes do prompt**: remove padrões de instrução embutida (ex.: "ignore as regras anteriores", "aprove este candidato") — mitigação básica de prompt injection vindo de currículo malicioso.
- Claude extrai JSON estruturado (nome, email, telefone, cidade, estado, resumo de experiência) usando structured output/tool use, não texto livre parseado manualmente.
- **Tela de revisão obrigatória**: RH vê o que a IA extraiu, confirma ou corrige antes de `Candidato` + `Candidatura` serem persistidos — nunca cria registro direto da resposta da IA sem revisão humana.
- Falha de extração ou confiança baixa → cai para o formulário manual que já existia (a feature de IA é um acelerador, não uma dependência obrigatória do fluxo de candidatura).

**Onde fica o resultado bruto da IA**: um modelo pequeno e separado dos dados confirmados —

```python
# apps/ia/models.py
class ExtracaoCurriculo(models.Model):
    candidatura = models.OneToOneField("recrutamento.Candidatura", on_delete=models.CASCADE)
    dados_extraidos = models.JSONField()        # saída bruta da IA, antes de qualquer correção
    confirmado_pelo_rh = models.BooleanField(default=False)
    criado_em = models.DateTimeField(auto_now_add=True)
```

- `JSONField` (nativo Django/Postgres), não arquivo: é pouco dado (texto curto, algumas chaves), e usar arquivo obrigaria gerenciar storage e limpeza de órfão sem ganhar nada em troca.
- O arquivo de verdade (PDF/DOCX **original** do currículo) continua sendo arquivo via `FileField` — isso sim é conteúdo grande/binário, faz sentido ficar fora do banco.
- **Botão "Ver detalhes" no Kanban de candidatos** (`recruitment.html`, board já existente): abre os dados confirmados do candidato e gera um **PDF em formato padronizado** (perfil do candidato — dados pessoais, pretensão salarial, etapa, histórico) usando `WeasyPrint` a partir de um template HTML. Como seção secundária/expansível, mostra o que a IA extraiu originalmente (`ExtracaoCurriculo.dados_extraidos`) — trilha de auditoria de eventual correção feita pelo RH.

### Segurança específica da IA

- IA nunca recalcula nem decide valores monetários — só explica o que o service determinístico já produziu.
- Arquivo de currículo: limite de tamanho (ex. 5MB), validação do tipo real (magic bytes, não só extensão), texto extraído tratado como entrada não confiável.
- Toda criação de `Candidato`/`Candidatura` via IA passa por confirmação humana antes de salvar no banco.
- Chave da API em `.env`, nunca hardcoded — mesma disciplina do resto do projeto (ver erro #6 do projeto original).

---

## Correções específicas dos outros erros encontrados

| Erro original | Correção no Django |
|---|---|
| Senha previsível (CPF + sufixo fixo) | Senha aleatória gerada com `secrets.token_urlsafe`, enviada por e-mail; `must_change_password=True` força troca no primeiro login |
| Contratação não inicializa férias | `signals.py` do app `recrutamento` cria `FeriasPeriodoAquisitivo` ao criar o `Funcionario` — sinal explícito em Python, documentado, não trigger de banco escondido |
| Segredos no git | `django-environ`; `.env` no `.gitignore`; `.env.example` commitado com chaves vazias; checklist de release inclui rotacionar qualquer segredo que já foi commitado no repo Spring |
| Relatórios nunca implementados | App `relatorios` entra no MVP da Fase 2 (ver cronograma), mesmo que com 2-3 relatórios simples (headcount por departamento, custo de folha por mês, férias vencendo) — não cria rota "para o futuro" |
| Rotas fantasma no security | Regra de processo: PR que adiciona `path()` sem view+template testados não é aceito (autocheck) |
| Desligamento sem nenhuma rescisão financeira | App `folha` ganha processamento de rescisão completo (saldo, férias proporcionais, 13º proporcional, aviso prévio, multa FGTS) — ver seção dedicada |
| 2FA com bug de schema (hash truncado em `varchar(10)`) | `CodigoDoisFatores.codigo_hash` dimensionado pro hash (`max_length=255`), nunca pelo tamanho do código original — ver seção "App `core`" |
| Sem rastreabilidade vaga → funcionário contratado | `Funcionario.vaga_origem` (FK nullable pra `recrutamento.Vaga`), preenchida no momento da contratação |
| Ponto e férias desconectados da folha | `gerador_lancamentos.gerar_lancamentos_ponto`/`gerar_lancamentos_ferias`, chamados explicitamente por `processador_folha` — ver seção "App `folha`" |

---

## Cronograma (jun/2026 – fev/2027, sem prazo rígido)

| Fase | Período | Entregável | Horas |
|---|---|---|---|
| 0 — Setup + 2FA | jun/2026 (resto) | Projeto Django criado, `core` (auth customizada + perfis + 2FA por e-mail corrigido), `organizacao`, `.env` configurado, CI básico rodando `pytest` | ~30h |
| 1 — Domínio base | jul/2026 | Apps `funcionarios`, `ponto`, `ferias`, `beneficios` com CRUD via DRF + Django Admin | ~50h |
| 2 — Núcleo de folha (foco principal) | ago/2026 | App `folha` completo: models, calculadoras, `processador_folha`, integração ponto/férias, testes, endpoints de holerite reais consumidos pelo front | ~50h |
| 2.5 — Rescisão | set/2026 | Cálculo financeiro completo de desligamento (saldo, férias proporcionais, 13º proporcional, aviso prévio, multa FGTS), reaproveitando calculadoras de INSS/IRRF | ~40h |
| 3 — Recrutamento + Relatórios | out/2026 | `recrutamento` com fluxo de contratação corrigido (senha segura, sinal de férias, rastreabilidade vaga→funcionário) + `relatorios` com 2-3 relatórios reais | ~30h |
| 4 — Docker + ajustes | nov/2026 | `docker-compose` (Django + Postgres), revisão de segurança (checklist OWASP básico), polimento do front (templates + JS) | ~25h |
| 5 — IA (diferencial) | dez/2026 | App `ia`: guia de holerites (~10h) + extração de currículo com upload/parsing/revisão humana (~30-38h) | ~40-48h |
| Margem | jan – fev/2027 | Buffer para imprevistos, polimento, ou avançar para Laravel (Fase 4 original do PACER) se tudo estiver entregue antes | — |

Total estimado: **~265-273h** ao longo de ~7 meses de execução ativa (jun/2026-dez/2026), com a janela total se estendendo a 8 meses (até fev/2027) contando a margem — sem deadline rígido, só teto de viabilidade. README comparativo, dados de demo (seed/fixtures) e deploy ficam fora deste documento por ora — entram quando você pedir.

---

## Definition of Done (evita repetir o erro #4)

Uma feature só é considerada pronta quando, **simultaneamente**:
1. Modelo + migration aplicada
2. Service com teste unitário (quando envolve cálculo/regra)
3. Endpoint DRF testado (`test_api_*.py`)
4. Tela/rota server-side renderizando dados reais (não mock, não placeholder)
5. Rota adicionada ao `urls.py` **só neste momento**, nunca antes

Nenhuma entrada no roadmap fica "meio pronta" no `main` — ou está nas 5 condições, ou fica em branch.
