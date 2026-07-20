# Segurança e LGPD

## O que é
Práticas pra proteger o sistema (contra ataques) e os **dados pessoais** (conformidade com a LGPD — Lei Geral de Proteção de Dados). Um sistema de RH lida com CPF, salário, endereço: dado sensível.

## Por que neste projeto
- O original **vazou segredos no git** e gerava **senha previsível** — erros que não podem se repetir.
- Há **uso real esperado** (D-15), então segurança deixa de ser opcional.
- A IA processa currículos (entrada não confiável) — risco de **prompt injection** (ver [[IA com a Claude API]]).

## Fundamentos a dominar
- **Autenticação × Autorização**: quem é você (login, [[Autenticação e 2FA]]) × o que você pode (policies, [[Perfis de Acesso]]).
- **Segredos**: `.env` fora do git, `.env.example` com chaves vazias, rotacionar o que já vazou.
- **Hash de senha**: bcrypt/argon (`Hash::make`), nunca guardar senha em texto puro. Por que dimensionar o campo pro hash (o bug do 2FA, D-11).
- **OWASP Top 10** (nível introdutório): SQL injection (o Eloquent protege), XSS (o Blade escapa por padrão), CSRF (token do Laravel), controle de acesso quebrado.
- **Validação de upload**: tamanho, tipo real por **magic bytes** (não só extensão) — usado no currículo.
- **LGPD na prática**: minimização de dados, **auditoria de acesso/alteração** (por isso o `spatie/laravel-activitylog`, D-13), direito de saber quem mexeu no dado.
- **Prompt injection**: por que sanitizar o texto do currículo antes de mandar pra IA.

## Onde estudar
- **OWASP Top 10** (owasp.org/www-project-top-ten) — a referência de vulnerabilidades.
- **Laravel docs → "Security"** (Authentication, Authorization, Encryption, Hashing, CSRF).
- Portal oficial da **LGPD** (gov.br/anpd) pra os princípios da lei.

## Como saber que entendeu
Consegue apontar, dos erros do projeto original (senha previsível, segredo no git, hash truncado), como cada um é corrigido — e explicar por que o currículo é "entrada não confiável".
