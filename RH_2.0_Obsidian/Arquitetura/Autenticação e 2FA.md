# Autenticação e 2FA

Domínio `Core`. Ver decisão D-11 em [[Registro de Decisões]].

## Login (SaaS multitenant)

- **Sem "cadastro" público.** O e-mail é criado na **contratação**: `joão.jerosch@{tag-da-empresa}.com.br`.
- Tela de login tem só **entrar** + **esqueci minha senha**.
- Senha inicial aleatória (`Str::random(32)`) enviada por e-mail; `must_change_password=true` força a troca no 1º login (corrige o bug do original, que gerava senha previsível a partir do CPF).

## 2FA por e-mail

Arquitetura correta herdada do Java: **gerar código → enviar e-mail → bloquear rota até validar**. O bug original era só de schema (hash de ~60 chars numa coluna `varchar(10)`).

### Model `CodigoDoisFatores`
```
codigo_hash  string(255)   # hash, nunca o código puro — dimensionado pro hash
expira_em    timestamp
usado        boolean
tentativas   smallint
```
> Regra que virou lei: **nunca dimensionar campo de hash pelo tamanho do dado original.**

### Service `DoisFatoresService`
- `gerarCodigo()` — 6 dígitos com `random_int()`, guarda `Hash::make()`, invalida códigos anteriores, envia e-mail.
- `validarCodigo()` — pega o código válido mais recente, compara com `Hash::check()`, incrementa tentativas, bloqueia após o limite.

### Parâmetros definidos (Q2)
| Parâmetro | Valor |
|---|---|
| Tentativas até bloquear | **5** |
| Expiração do código | **10 minutos** |
| Reenvio | até **5x**; depois, esperar cooldown |
| Bloqueio | **cooldown progressivo**: 1º = 1 dia, 2º = 2 dias, 3º = 4 dias |

### Middleware `ExigeDoisFatores`
Redireciona pra tela de 2FA enquanto a sessão não tiver `2fa_verificado`, exceto nas rotas de login/2fa.

## Testes obrigatórios
Gerar código e confirmar que o hash cabe no campo; código expirado, usado e errado; bloqueio após 5 tentativas; cooldown progressivo.

## Ligações
Perfis que o login habilita: [[Perfis de Acesso]]. Segurança geral: [[Segurança e LGPD]].
