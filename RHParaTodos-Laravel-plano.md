# RHParaTodos-Laravel — Plano Técnico de Reconstrução

> Este documento substitui `RHParaTodos-Django-plano.md` como plano principal. O arquivo Django é mantido intacto como histórico da decisão anterior — não foi apagado nem editado.

## Contexto

O RHParaTodos original (Java/Spring Boot + Thymeleaf + PostgreSQL) ficou incompleto e com funcionalidades confusas. Auditoria do projeto em `C:\Users\joaoj\Documents\GitHub\RHParaTodos` encontrou:

1. **Cálculo de holerite (INSS/IRRF/FGTS) existe só no banco**, em ~15 functions/triggers PL/pgSQL (`V1__baseline.sql`), e nunca é chamado pela aplicação Java — zero entidade/service/controller toca essas tabelas.
2. **Tela de holerites é 100% fake**: `payslips.js` e `employee.js` geram dados com `Math.random()` em `localStorage`, sem nenhuma chamada de API. Há duas implementações mock divergentes da mesma feature.
3. **Rota `/payslips` quebrada**: o controller mapeia para uma view que não existe nos templates (erro 500 se acessada).
4. **`SecurityConfig` referencia módulos nunca implementados**: `/payroll/**`, `/reports/**`, `/performance/**`, `/training/**` têm regra de autorização mas nenhum controller — projeto "esqueletado" maior do que o entregue.
5. **Contratação automática gera senha previsível** (6 dígitos do CPF + sufixo fixo) e não inicializa férias do novo funcionário.
6. **Segredos reais commitados no git** (senha de app do Gmail, JWT secret) em `application.properties`, rastreados desde commits antigos.

Este documento define como reconstruir o sistema em **Laravel**, evitando repetir esses erros — em especial, **toda regra de negócio de folha de pagamento vive em código PHP testável, nunca em função/trigger de banco**.

**Janela de tempo**: 7-8 meses, sem prazo rígido — 6 meses era só o teto de viabilidade inicial. **Stack travada: Laravel + Blade + JS vanilla, sem React por agora.**

**Diferencial de IA**: o projeto inclui duas features de inteligência artificial (Claude API) como diferencial de portfólio — guia de holerites em linguagem natural e extração automática de dados de currículo. Ver seção dedicada "Domain `Ia`".

**Mudança de stack em relação ao plano anterior**: o projeto seria feito em Django, mas a decisão final foi reconstruir tudo em Laravel/PHP. A arquitetura de negócio (nada de trigger no banco, services testáveis, rescisão completa, IA com revisão humana) é a mesma — só a implementação técnica muda. As diferenças relevantes estão marcadas ao longo do documento.

---

## Decisão de arquitetura: por que nada de trigger/function no banco

O banco (PostgreSQL) fica restrito a:
- Schema (tabelas, colunas, tipos)
- Constraints de integridade: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK` simples (ex.: `valor >= 0`)
- Índices

Tudo que é **regra de negócio** — cálculo de INSS/IRRF progressivo, geração de lançamentos de folha, fechamento/reabertura de folha, elegibilidade de férias, etc. — vive em **services PHP puros**, com o mínimo de acoplamento ao Eloquent dentro da lógica de cálculo (facilita testar com PHPUnit/Pest sem precisar montar cenário de banco completo). Motivo prático: tabela de IRRF/INSS muda todo ano por lei — isso precisa estar em código revisável por PR e coberto por teste unitário, não escondido em SQL que ninguém revisa.

Justificativa adicional, ligada ao erro #4 do projeto anterior: **nenhuma rota entra em `routes/web.php`/`routes/api.php` sem o endpoint e a tela atrás dela já funcionando**. Isso é regra de processo, não só de código — ver seção "Definition of Done" no final.

---

## Stack

| Camada | Escolha | Observação |
|---|---|---|
| Backend | **Laravel 11.x** | Rotas/controllers REST já nativos — não precisa de framework de API separado (papel do DRF no plano Django) |
| Banco | **PostgreSQL** | Mesmo motor do projeto original e do plano Django — facilita comparar os três |
| Auth API | **Laravel Sanctum** | Tokens de API, equivalente ao `simplejwt` do plano anterior |
| Auth web | Sessão Laravel (guard `web`) | Mesmo padrão dual (API + sessão) do plano anterior |
| Front | **Blade** + JS vanilla (`fetch`) | Mesma filosofia do plano anterior (templates server-side + JS vanilla pontual), só troca de sintaxe de template |
| Painel admin RH/DP | **Filament** | Ver seção dedicada — equivalente funcional ao Django Admin que o plano anterior usava na Fase 1 |
| Config/segredos | `.env` nativo do Laravel | Não precisa de pacote equivalente ao `django-environ`, já vem de fábrica |
| Testes | **PHPUnit** ou **Pest** (decisão: Pest, sintaxe mais legível) | Equivalente ao `pytest-django` |
| Migrations | Migrations nativas do Laravel | Substituem o Flyway, mesmo papel que tinham no plano Django |
| Precisão decimal | **`brick/math`** (`BigDecimal`) | **Ponto crítico**: PHP não tem `Decimal` nativo como Python. Usar `float` em cálculo de folha é inaceitável (erro de arredondamento). `brick/math` dá objetos imutáveis de precisão arbitrária, papel equivalente ao `decimal.Decimal` do plano anterior |
| IA | Wrapper próprio sobre **Guzzle** (cliente HTTP) contra a API REST da Claude | **Não existe SDK oficial em PHP da Anthropic** (só Python/TypeScript/Java/Go/Ruby) — diferença real de esforço em relação ao plano anterior, que usava o SDK oficial `anthropic` |
| Extração de texto de currículo | `smalot/pdfparser` (PDF) + `PhpOffice/PhpWord` (DOCX) | Equivalentes a `pdfplumber`/`python-docx`, ecossistema PHP menos maduro nessa área |
| PDF (holerite, perfil do candidato) | **`spatie/laravel-pdf`** (Browsershot/Chromium) | Escolhido sobre `barryvdh/laravel-dompdf` por ter suporte completo a CSS moderno (igual o `WeasyPrint` do plano anterior tinha) — custo: depende de Node/Chromium instalado no ambiente |

---

## Estrutura de módulos (Laravel não tem "apps" nativos)

Laravel não tem o conceito de apps isolados que o Django tem. Decisão: replicar a separação por domínio manualmente, sem depender de pacote de terceiros (`nwidart/laravel-modules` foi considerado e descartado — adiciona service providers e configuração de rotas por módulo que não trazem benefício real num projeto de um único desenvolvedor).

```
app/
├── Domain/
│   ├── Core/                  # Usuario, autenticação, perfis, 2FA por e-mail (ver seção dedicada)
│   │   ├── Models/
│   │   └── Services/
│   ├── Organizacao/            # Departamento, Cargo
│   ├── Funcionarios/           # Funcionario, Promocao
│   ├── Recrutamento/           # Vaga, Candidato, Candidatura
│   ├── Ponto/                   # PontoMarcacao, PontoApuracaoDiaria, PontoCalendario, PontoOcorrencia
│   ├── Ferias/                  # FeriasPeriodoAquisitivo, FeriasSolicitacao, FeriasRegistro
│   ├── Beneficios/              # TipoBeneficio, FuncionarioBeneficio
│   ├── Folha/                   # 🔑 núcleo da reconstrução — ver seção dedicada
│   │   └── Services/
│   │       └── Rescisao/
│   ├── Solicitacoes/            # Solicitacao, StatusSolicitacao, TipoSolicitacao
│   ├── Relatorios/              # módulo que NUNCA existiu no projeto original — entra desde o início
│   └── Ia/                      # 🤖 diferencial — guia de holerites + extração de currículo
├── Filament/                   # Resources do painel admin (RH/DP) — telas internas de CRUD
└── Http/Controllers/Api/        # Controllers REST que expõem os domínios acima
```

Cada domínio = mesma divisão do projeto Spring e do plano Django (`Funcionario`, `Vaga`, `Candidatura`, etc.), só que sem o vácuo entre "tabela existe" e "regra de negócio existe".

---

## Painel administrativo (RH/DP): Filament

O plano Django usava o Django Admin pronto pra CRUD interno (`funcionarios`, `ponto`, `ferias`, `beneficios` — ver Fase 1 do cronograma daquele plano) e reservava telas customizadas só para fluxos voltados a funcionário/candidato (`meus-holerites`, Kanban de recrutamento). Laravel não tem painel administrativo nativo — a escolha equivalente é o **Filament**:

- **Filament Resources** cobrem CRUD interno de `organizacao`, `funcionarios`, `ponto`, `ferias`, `beneficios`, `solicitacoes` e a visualização de `relatorios` — mesmo papel que o Django Admin tinha.
- **Blade + JS vanilla customizado** continua exclusivo para as telas voltadas a funcionário/candidato: `meus-holerites`, Kanban de candidatos (`recruitment`), tela de revisão humana da extração de currículo por IA.
- Custo: curva de aprendizado do Filament entra no orçamento de horas da Fase 1 (ver cronograma).

---

## Domain `Core` — autenticação e 2FA por e-mail

No projeto original, o 2FA por e-mail estava **bem desenhado, mas com um bug de schema fatal**: `TwoFactorCodeStore.java` inseria o hash bcrypt do código (~60 caracteres) numa coluna `varchar(10)` — truncava e quebrava em runtime sempre que um código era gerado. Fora esse bug, o fluxo (gerar código → enviar e-mail → bloquear rotas até validar) estava certo. A reconstrução mantém a mesma arquitetura, só corrigindo a causa raiz: nunca dimensionar um campo de hash pelo tamanho do dado original.

### Migration + Model

```php
// database/migrations/..._create_codigos_dois_fatores_table.php
Schema::create('codigos_dois_fatores', function (Blueprint $table) {
    $table->id();
    $table->foreignId('usuario_id')->constrained('usuarios')->cascadeOnDelete();
    $table->string('codigo_hash', 255); // hash, nunca o código puro — dimensionado pro hash, não pro código de 6 dígitos
    $table->timestamp('expira_em');
    $table->boolean('usado')->default(false);
    $table->unsignedSmallInteger('tentativas')->default(0);
    $table->timestamps();
});
```

```php
// app/Domain/Core/Models/CodigoDoisFatores.php
class CodigoDoisFatores extends Model
{
    protected $table = 'codigos_dois_fatores';
    protected $fillable = ['usuario_id', 'codigo_hash', 'expira_em', 'usado', 'tentativas'];
    protected $casts = ['expira_em' => 'datetime', 'usado' => 'boolean'];

    public function usuario(): BelongsTo
    {
        return $this->belongsTo(Usuario::class);
    }
}
```

### Service (`app/Domain/Core/Services/DoisFatoresService.php`)

- `gerarCodigo(Usuario $usuario)`: gera 6 dígitos com `random_int()` (criptograficamente seguro em PHP, equivalente ao `secrets.randbelow` do Python), guarda o hash via `Hash::make()` (bcrypt, reaproveita a infra nativa do Laravel), define expiração curta (ex.: 5 min), invalida códigos anteriores não usados do mesmo usuário, envia e-mail via `Mail::send()` com uma classe `Mailable`.
- `validarCodigo(Usuario $usuario, string $codigoInformado)`: busca o código válido mais recente (não expirado, não usado), compara com `Hash::check()`, incrementa `tentativas` e bloqueia depois de N tentativas erradas, marca como usado se válido.

```php
class DoisFatoresService
{
    public function gerarCodigo(Usuario $usuario): void
    {
        CodigoDoisFatores::where('usuario_id', $usuario->id)->where('usado', false)->update(['usado' => true]);

        $codigo = (string) random_int(100000, 999999);

        CodigoDoisFatores::create([
            'usuario_id' => $usuario->id,
            'codigo_hash' => Hash::make($codigo),
            'expira_em' => now()->addMinutes(5),
        ]);

        Mail::to($usuario->email)->send(new CodigoDoisFatoresMail($codigo));
    }

    public function validarCodigo(Usuario $usuario, string $codigoInformado): bool
    {
        $registro = CodigoDoisFatores::where('usuario_id', $usuario->id)
            ->where('usado', false)
            ->where('expira_em', '>', now())
            ->latest()
            ->first();

        if (!$registro || $registro->tentativas >= 5) {
            return false;
        }

        if (!Hash::check($codigoInformado, $registro->codigo_hash)) {
            $registro->increment('tentativas');
            return false;
        }

        $registro->update(['usado' => true]);
        return true;
    }
}
```

Teste dedicado que o projeto original nunca teve: gerar um código e confirmar que o hash cabe no campo, testar código expirado, usado e errado, e testar o bloqueio após N tentativas.

### Middleware (equivalente ao `TwoFactorEnforcementFilter`)

```php
// app/Http/Middleware/ExigeDoisFatores.php
class ExigeDoisFatores
{
    public function handle(Request $request, Closure $next)
    {
        if (!$request->session()->get('2fa_verificado') && !$request->routeIs('login', '2fa.*')) {
            return redirect()->route('2fa.form');
        }

        return $next($request);
    }
}
```

---

## Domain `Folha` — núcleo da reconstrução

### Models (apenas dados, sem lógica)

```php
// app/Domain/Folha/Models/FolhaPagamento.php
class FolhaPagamento extends Model
{
    protected $fillable = ['funcionario_id', 'competencia', 'tipo', 'status', 'total_proventos', 'total_descontos', 'valor_liquido', 'fechada_em', 'fechada_por'];
    protected $casts = ['competencia' => 'date', 'fechada_em' => 'datetime'];

    public function lancamentos(): HasMany
    {
        return $this->hasMany(LancamentoFolha::class);
    }
}
```

```php
// app/Domain/Folha/Models/LancamentoFolha.php
class LancamentoFolha extends Model
{
    protected $fillable = ['folha_pagamento_id', 'tipo', 'codigo', 'descricao', 'referencia', 'valor', 'origem'];
}
```

Migrations equivalentes para `TabelaIRRF`, `ParametroIRRF`, `TabelaINSS` — mesmos campos do plano anterior (`vigente_desde`, `faixa_min`, `faixa_max`, `aliquota`, `parcela_deduzir`). Sem nenhum `DB::unprepared()` de function/trigger. Tudo que era `CREATE FUNCTION` no projeto Spring se torna uma das classes abaixo.

### Services (regra de negócio, 100% PHP, 100% testável)

```
app/Domain/Folha/Services/
├── CalculadoraInss.php       # calcular(BigDecimal $base, int $ano): BigDecimal
├── CalculadoraIrrf.php       # calcular(BigDecimal $baseLiquida, Carbon $dataReferencia): BigDecimal
├── CalculadoraFgts.php       # calcular(BigDecimal $base): BigDecimal
├── GeradorLancamentos.php    # monta os LancamentoFolha (salário, benefícios, horas extra/falta vindas do ponto, férias)
└── ProcessadorFolha.php      # orquestra: abrir folha -> gerar lançamentos -> calcular impostos -> totalizar -> fechar
```

**Achado da auditoria que motiva o design abaixo**: no projeto original, `FeriasService` e `PontoService` nunca chamavam as functions de folha — não é só a folha que estava isolada, é **todo módulo que deveria alimentar a folha que estava desconectado dela**. Pra não repetir isso, a integração é explícita e tem dono:

```php
// app/Domain/Folha/Services/GeradorLancamentos.php
class GeradorLancamentos
{
    /** Lê PontoApuracaoDiaria do período e cria lançamentos HEXTRA/FALTA. Chamado explicitamente por ProcessadorFolha. */
    public function gerarLancamentosPonto(FolhaPagamento $folha, Carbon $competencia): void { /* ... */ }

    /** Cria lançamentos FERIAS/FERIAS_1_3 quando uma FeriasRegistro é aprovada pra competência da folha aberta. */
    public function gerarLancamentosFerias(FolhaPagamento $folha, FeriasRegistro $feriasRegistro): void { /* ... */ }
}
```

Cada método chamado de forma explícita por `ProcessadorFolha` (ou por um Eloquent Observer documentado em comentário, nunca implícito) e com teste próprio — exatamente o ponto onde o projeto original quebrou silenciosamente.

Exemplo de contrato (espelha 1:1 o que a function `calcular_irrf_progressivo` fazia no SQL, só que testável):

```php
// app/Domain/Folha/Services/CalculadoraIrrf.php
use Brick\Math\BigDecimal;
use Brick\Math\RoundingMode;

class CalculadoraIrrf
{
    public function calcular(BigDecimal $baseLiquida, Carbon $dataReferencia): BigDecimal
    {
        $faixa = TabelaIrrf::where('vigente_desde', '<=', $dataReferencia)
            ->where('faixa_min', '<=', (string) $baseLiquida)
            ->where(fn ($q) => $q->where('faixa_max', '>=', (string) $baseLiquida)->orWhereNull('faixa_max'))
            ->orderByDesc('vigente_desde')
            ->first();

        if (!$faixa) {
            return BigDecimal::zero();
        }

        $valor = $baseLiquida
            ->multipliedBy($faixa->aliquota)
            ->minus($faixa->parcela_deduzir)
            ->toScale(2, RoundingMode::HALF_UP);

        return $valor->isNegative() ? BigDecimal::zero() : $valor;
    }
}
```

```php
// app/Domain/Folha/Services/ProcessadorFolha.php
class FolhaJaFechadaException extends \Exception {}

class ProcessadorFolha
{
    public function __construct(
        private CalculadoraInss $calculadoraInss,
        private CalculadoraIrrf $calculadoraIrrf,
        private CalculadoraFgts $calculadoraFgts,
    ) {}

    public function processar(int $folhaId): FolhaPagamento
    {
        return DB::transaction(function () use ($folhaId) {
            $folha = FolhaPagamento::lockForUpdate()->findOrFail($folhaId);

            if ($folha->status === 'FECHADA') {
                throw new FolhaJaFechadaException("Folha {$folhaId} está fechada.");
            }

            $folha->lancamentos()->where('origem', 'CALCULO_AUTOMATICO')->delete();

            $baseInss = $this->baseInss($folha);
            $valorInss = $this->calculadoraInss->calcular($baseInss, $folha->competencia->year);
            $this->registrarLancamento($folha, 'DESCONTO', 'INSS', $baseInss, $valorInss);

            $baseIrrf = $this->baseIrrf($folha)->minus($valorInss);
            $valorIrrf = $this->calculadoraIrrf->calcular($baseIrrf, $folha->competencia);
            $this->registrarLancamento($folha, 'DESCONTO', 'IRRF', $baseIrrf, $valorIrrf);

            $valorFgts = $this->calculadoraFgts->calcular($this->baseFgts($folha));
            $this->registrarLancamento($folha, 'INFORMATIVO', 'FGTS', $this->baseFgts($folha), $valorFgts);

            $this->recalcularTotais($folha);

            return $folha;
        });
    }
}
```

Cada classe (`CalculadoraInss`, `CalculadoraIrrf`, `CalculadoraFgts`, `ProcessadorFolha`) ganha teste unitário (Pest) com tabelas de exemplo — isso é o que o projeto anterior nunca teve e que o banco não permitia ter de forma simples.

### Testes mínimos obrigatórios (`tests/Unit/Folha/`)

- `CalculadoraIrrfTest`: cada faixa da tabela progressiva, valores de borda (exatamente no limite de faixa), base zero, base negativa (deve recusar ou zerar).
- `CalculadoraInssTest`: idem, progressivo por faixa.
- `ProcessadorFolhaTest`: fluxo completo abrir → processar → fechar; tentar reprocessar folha fechada deve lançar `FolhaJaFechadaException`; reabrir folha permite reprocessar.
- `ApiFolhaTest` (feature test): endpoint de geração de holerite retorna os mesmos valores calculados pelos services (sem duplicar lógica no controller).

---

## API — só entra rota com tela funcionando atrás

Mesma exigência que faltou no projeto original (`/payroll`, `/reports` existiam no Security mas não no resto):

| Recurso | Endpoint | Observação |
|---|---|---|
| Folha do funcionário | `GET /api/folha/meus-holerites` | Substitui o mock do `payslips.js` — dados reais vindos de `LancamentoFolha` |
| Detalhe de um holerite | `GET /api/folha/{id}` | Inclui lista de lançamentos (proventos/descontos) |
| Processar folha (RH/DP) | `POST /api/folha/{id}/processar` | Chama `ProcessadorFolha::processar()`, nunca SQL direto |
| Fechar folha | `POST /api/folha/{id}/fechar` | Bloqueia novos lançamentos após fechada (checagem em PHP, não trigger) |
| Reabrir folha | `POST /api/folha/{id}/reabrir` | Só ADMIN/DP_CHEFE |
| PDF do holerite | `GET /api/folha/{id}/pdf` | Gera o PDF **dinamicamente a cada request** — ver decisão abaixo |

### Geração do PDF do holerite: dinâmico, sem armazenar arquivo

Mesma decisão do plano anterior: o PDF **não é gerado uma vez e guardado** — é renderizado on-demand a cada download, direto dos `LancamentoFolha` daquela `FolhaPagamento` via `spatie/laravel-pdf`.

- A fonte de verdade é imutável depois que a folha está `FECHADA` (`ProcessadorFolha` já bloqueia reprocessamento via `FolhaJaFechadaException`), então renderizar na hora sempre reproduz o mesmo conteúdo financeiro.
- Evita depender de storage em nuvem sem necessidade real nesse estágio do projeto.
- Mais simples de testar: só precisa validar a renderização do template Blade com dados fixos.
- Ressalva conhecida (igual ao plano anterior): se o layout do template mudar no futuro, um holerite antigo reimpresso herda a aparência nova — os valores continuam corretos, só a formatação visual não é "congelada".

---

## Domain `Folha` — rescisão (desligamento com cálculo financeiro completo)

**Achado da auditoria**: no projeto original, desligar um funcionário só muda `status` para `INATIVO` e grava `dataDesligamento` — zero cálculo de saldo de salário, férias proporcionais, 13º proporcional, aviso prévio ou multa de FGTS. A reconstrução trata rescisão como um processamento de folha de verdade, reaproveitando as calculadoras de INSS/IRRF já existentes.

### Migration (campos novos em `funcionarios`)

```php
Schema::table('funcionarios', function (Blueprint $table) {
    $table->enum('tipo_desligamento', ['SEM_JUSTA_CAUSA', 'PEDIDO_DEMISSAO', 'JUSTA_CAUSA', 'ACORDO'])->nullable();
    $table->enum('aviso_previo_tipo', ['INDENIZADO', 'TRABALHADO', 'DISPENSADO'])->nullable();
});
```

`FolhaPagamento.tipo` ganha um valor `RESCISAO` (além do mensal padrão) — reaproveita a mesma estrutura de `LancamentoFolha`, PDF e explicação por IA já desenhada pra folha mensal.

### Services (`app/Domain/Folha/Services/Rescisao/`)

```
app/Domain/Folha/Services/Rescisao/
├── CalculadoraSaldoSalario.php            # dias trabalhados no mês / 30 * salário
├── CalculadoraFeriasRescisao.php          # férias vencidas (se houver) + proporcionais, ambas com 1/3
├── CalculadoraDecimoTerceiroProporcional.php
├── CalculadoraAvisoPrevio.php             # indenizado vs trabalhado vs dispensado
├── CalculadoraMultaFgts.php               # 40% (sem justa causa) ou 20% (acordo) sobre o saldo de FGTS depositado
├── RegrasPorTipoDesligamento.php          # tabela explícita: o que cada tipo de desligamento perde/mantém direito
└── ProcessadorRescisao.php                # orquestra tudo, gera FolhaPagamento tipo RESCISAO
```

`RegrasPorTipoDesligamento` é o ponto mais sensível — cada tipo de desligamento muda quais verbas se aplicam. Fica como uma estrutura de dados explícita e testável, não um `if/else` espalhado:

```php
// app/Domain/Folha/Services/Rescisao/RegrasPorTipoDesligamento.php
class RegrasPorTipoDesligamento
{
    public const REGRAS = [
        'SEM_JUSTA_CAUSA' => ['ferias_proporcionais' => true,  'decimo_proporcional' => true,  'aviso_previo_indenizado' => true,  'multa_fgts_pct' => 40],
        'PEDIDO_DEMISSAO' => ['ferias_proporcionais' => true,  'decimo_proporcional' => true,  'aviso_previo_indenizado' => false, 'multa_fgts_pct' => 0],
        'JUSTA_CAUSA'     => ['ferias_proporcionais' => false, 'decimo_proporcional' => false, 'aviso_previo_indenizado' => false, 'multa_fgts_pct' => 0],
        'ACORDO'          => ['ferias_proporcionais' => true,  'decimo_proporcional' => true,  'aviso_previo_indenizado' => true,  'multa_fgts_pct' => 20],
    ];
}
```

`ProcessadorRescisao::processar(int $funcionarioId, string $tipoDesligamento, Carbon $dataDesligamento, ?string $avisoPrevioTipo)`:
1. Consulta `RegrasPorTipoDesligamento::REGRAS[$tipoDesligamento]` pra saber quais verbas calcular.
2. Calcula saldo de salário, férias (vencidas + proporcionais), 13º proporcional, aviso prévio — só os que a regra permite.
3. Aplica `CalculadoraInss`/`CalculadoraIrrf` (reaproveitados da folha mensal) sobre saldo de salário e 13º separadamente — são bases de cálculo distintas.
4. Calcula multa de FGTS sobre o histórico de FGTS depositado (soma dos lançamentos `FGTS` em `LancamentoFolha` ao longo do vínculo).
5. Gera `FolhaPagamento` tipo `RESCISAO` com todos os lançamentos, fecha automaticamente.

### Testes obrigatórios

- Cada calculadora isolada (saldo, férias rescisórias, 13º proporcional, aviso prévio, multa FGTS).
- `RegrasPorTipoDesligamentoTest`: confirma que justa causa não gera férias/13º proporcional nem aviso prévio indenizado; que pedido de demissão não gera multa FGTS; etc.
- `ProcessadorRescisaoTest`: cenário completo de demissão sem justa causa, ponta a ponta.

---

## Domain `Ia` — diferenciais de inteligência artificial

Centraliza tudo que chama a API da Anthropic (Claude), pra não espalhar chave/cliente HTTP pelo projeto. **Diferença chave em relação ao plano Django**: não existe SDK oficial em PHP, então o cliente é um wrapper fino sobre Guzzle direto contra a REST API.

```
app/Domain/Ia/
├── ClaudeClient.php                  # wrapper fino sobre Guzzle (model, timeout, retries)
├── Services/
│   ├── ExplicadorHolerite.php        # explica em português um holerite JÁ calculado
│   └── ExtratorCurriculo.php         # extrai texto de PDF/DOCX e pede JSON estruturado à IA
└── Models/
    └── ExtracaoCurriculo.php
```

```php
// app/Domain/Ia/ClaudeClient.php
class ClaudeClient
{
    public function __construct(private \GuzzleHttp\Client $http) {}

    public function mensagem(array $messages, ?array $tools = null): array
    {
        $response = $this->http->post('https://api.anthropic.com/v1/messages', [
            'headers' => [
                'x-api-key' => config('services.anthropic.key'),
                'anthropic-version' => '2023-06-01',
                'content-type' => 'application/json',
            ],
            'json' => array_filter([
                'model' => config('services.anthropic.model'),
                'max_tokens' => 1024,
                'messages' => $messages,
                'tools' => $tools,
            ]),
        ]);

        return json_decode($response->getBody()->getContents(), true);
    }
}
```

### 1. Guia de holerites

- Recebe a `FolhaPagamento` **já fechada** + seus `LancamentoFolha` — a IA nunca calcula nada, só explica números que `ProcessadorFolha` já produziu.
- Prompt restrito: passa os lançamentos como dado estruturado e pede explicação em português simples, com instrução explícita de não declarar nenhum valor que não esteja na lista recebida.
- Endpoint: `POST /api/folha/{id}/explicar`. Resposta cacheada (model `ExplicacaoHolerite` ou cache do Laravel) — folha fechada não muda, não precisa gerar de novo a cada clique.
- Botão "Explicar com IA" na tela de holerite do funcionário (`meus-holerites`).

### 2. Extração de dados de currículo

- Upload de PDF/DOCX na tela pública/RH de candidatura.
- `smalot/pdfparser` (PDF) / `PhpOffice/PhpWord` (DOCX) extraem o texto bruto.
- Texto **sanitizado antes do prompt**: remove padrões de instrução embutida — mitigação básica de prompt injection vindo de currículo malicioso.
- Claude extrai JSON estruturado (nome, email, telefone, cidade, estado, resumo de experiência) usando tool use/structured output, não texto livre parseado manualmente.
- **Tela de revisão obrigatória**: RH vê o que a IA extraiu, confirma ou corrige antes de `Candidato` + `Candidatura` serem persistidos — nunca cria registro direto da resposta da IA sem revisão humana.
- Falha de extração ou confiança baixa → cai para o formulário manual que já existia.

```php
// app/Domain/Ia/Models/ExtracaoCurriculo.php
class ExtracaoCurriculo extends Model
{
    protected $fillable = ['candidatura_id', 'dados_extraidos', 'confirmado_pelo_rh'];
    protected $casts = ['dados_extraidos' => 'array', 'confirmado_pelo_rh' => 'boolean'];

    public function candidatura(): BelongsTo
    {
        return $this->belongsTo(Candidatura::class);
    }
}
```

- Coluna `json` nativa (Postgres + cast `array` do Eloquent) — pouco dado, sem necessidade de arquivo.
- O arquivo de verdade (PDF/DOCX original do currículo) continua via `Storage` facade — isso sim é conteúdo grande/binário, fica fora do banco.
- **Botão "Ver detalhes" no Kanban de candidatos**: abre os dados confirmados do candidato e gera um PDF padronizado (perfil do candidato) via `spatie/laravel-pdf`, mostrando como seção secundária o que a IA extraiu originalmente — trilha de auditoria de eventual correção feita pelo RH.

### Segurança específica da IA

- IA nunca recalcula nem decide valores monetários — só explica o que o service determinístico já produziu.
- Arquivo de currículo: limite de tamanho (ex. 5MB), validação do tipo real (magic bytes, não só extensão), texto extraído tratado como entrada não confiável.
- Toda criação de `Candidato`/`Candidatura` via IA passa por confirmação humana antes de salvar no banco.
- Chave da API em `.env`, nunca hardcoded.

---

## Correções específicas dos outros erros encontrados

| Erro original | Correção em Laravel |
|---|---|
| Senha previsível (CPF + sufixo fixo) | Senha aleatória gerada com `Str::random(32)`, enviada por e-mail; `must_change_password=true` força troca no primeiro login |
| Contratação não inicializa férias | Eloquent Observer no domínio `Recrutamento` cria `FeriasPeriodoAquisitivo` ao criar o `Funcionario` — observer explícito, documentado, não trigger de banco escondido |
| Segredos no git | `.env` no `.gitignore`; `.env.example` commitado com chaves vazias; checklist de release inclui rotacionar qualquer segredo que já foi commitado no repo Spring |
| Relatórios nunca implementados | Domain `Relatorios` entra no MVP da Fase 2 (ver cronograma), mesmo que com 2-3 relatórios simples — não cria rota "para o futuro" |
| Rotas fantasma no security | Regra de processo: PR que adiciona rota sem controller+view testados não é aceito (autocheck) |
| Desligamento sem nenhuma rescisão financeira | Domain `Folha` ganha processamento de rescisão completo — ver seção dedicada |
| 2FA com bug de schema (hash truncado em `varchar(10)`) | `codigos_dois_fatores.codigo_hash` dimensionado pro hash (`string(255)`), nunca pelo tamanho do código original |
| Sem rastreabilidade vaga → funcionário contratado | `funcionarios.vaga_origem_id` (FK nullable pra `vagas`), preenchida no momento da contratação |
| Ponto e férias desconectados da folha | `GeradorLancamentos::gerarLancamentosPonto`/`gerarLancamentosFerias`, chamados explicitamente por `ProcessadorFolha` |

---

## Cronograma (sem prazo rígido, mesmo teto de viabilidade de 7-8 meses)

| Fase | Entregável | Horas | Observação vs. plano anterior |
|---|---|---|---|
| 0 — Setup + 2FA | Projeto Laravel criado, `Core` (auth customizada + perfis + 2FA por e-mail corrigido), `Organizacao`, `.env` configurado, CI básico rodando Pest | ~35h | +5h vs. plano Django: inclui revisão de fundamentos Laravel/Eloquent caso necessário |
| 1 — Domínio base | `Funcionarios`, `Ponto`, `Ferias`, `Beneficios` com CRUD via API + Filament | ~55-60h | +5-10h: curva de aprendizado do Filament (substitui o Django Admin "de graça") |
| 2 — Núcleo de folha (foco principal) | Domain `Folha` completo: models, calculadoras (com `brick/math`), `ProcessadorFolha`, integração ponto/férias, testes, endpoints de holerite reais | ~50-55h | +5h: setup de `brick/math` e disciplina de precisão decimal desde o início |
| 2.5 — Rescisão | Cálculo financeiro completo de desligamento, reaproveitando calculadoras de INSS/IRRF | ~40h | Sem mudança — lógica é a mesma, só sintaxe |
| 3 — Recrutamento + Relatórios | `Recrutamento` com fluxo de contratação corrigido + `Relatorios` com 2-3 relatórios reais | ~30h | Sem mudança |
| 4 — Docker + ajustes | `docker-compose` (Laravel + Postgres), revisão de segurança (checklist OWASP básico), polimento do front (Blade + JS) | ~25h | Sem mudança |
| 5 — IA (diferencial) | Domain `Ia`: guia de holerites (~10h) + extração de currículo com upload/parsing/revisão humana | ~45-55h | +5-7h: sem SDK oficial, o wrapper Guzyle/REST precisa de mais cuidado manual (retries, parsing de erro, tool use) do que usar o SDK Python pronto |
| Margem | Buffer para imprevistos, polimento | — | Sem mudança |

Total estimado: **~280-300h** (vs. ~265-273h do plano Django) — o aumento vem principalmente da ausência de painel admin nativo (Filament) e de SDK oficial de IA em PHP, não da lógica de negócio em si (que é idêntica). README comparativo, dados de demo (seed/fixtures) e deploy continuam fora deste documento por ora.

---

## Definition of Done (evita repetir o erro #4)

Uma feature só é considerada pronta quando, **simultaneamente**:
1. Migration aplicada + Model Eloquent
2. Service com teste unitário (Pest/PHPUnit) quando envolve cálculo/regra
3. Endpoint testado (feature test)
4. Tela Blade (ou Filament Resource, quando aplicável) renderizando dados reais — não mock, não placeholder
5. Rota adicionada a `routes/web.php`/`routes/api.php` **só neste momento**, nunca antes

Nenhuma entrada no roadmap fica "meio pronta" no `main` — ou está nas 5 condições, ou fica em branch.
