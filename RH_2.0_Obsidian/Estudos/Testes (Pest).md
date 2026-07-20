# Testes (Pest)

## O que é
**Testes automatizados** = código que verifica se o seu código está certo. **Pest** é o framework de testes escolhido (sintaxe mais limpa que o PHPUnit, sobre o qual ele roda).

## Por que neste projeto
É o que o projeto original **nunca teve** — e o que permite confiar na folha. Se um teste de IRRF passa em todas as faixas, você **sabe** que o cálculo está certo, sem conferir na mão. Testabilidade é critério de design aqui (ver [[Arquitetura de Software]]).

## Fundamentos a dominar
- **Tipos de teste**:
  - **Unitário** — testa uma classe isolada (ex.: `CalculadoraIrrf` com valores de borda). Rápido, sem banco.
  - **Feature** — testa um endpoint de ponta a ponta (ex.: `GET /api/folha/{id}` retorna os valores certos).
- **Estrutura AAA**: Arrange (prepara) → Act (executa) → Assert (verifica).
- **Assertions**: `expect($x)->toBe($y)`, `assertStatus(200)`, etc.
- **Casos de borda**: exatamente no limite de faixa, base zero, base negativa — foi onde o cálculo real quebra.
- **Database factories & migrations em teste** — criar dados falsos pra feature tests.
- **Rodar**: `php artisan test` ou `./vendor/bin/pest`.

## Onde estudar
- **Documentação do Pest** (pestphp.com/docs) — enxuta e clara.
- **Laravel docs → "Testing"** — como testar HTTP, banco, mail.
- Laracasts tem série de testes com Pest.

## Como saber que entendeu
Consegue escrever `CalculadoraIrrfTest` cobrindo cada faixa + bordas, e um feature test que garante que o controller não duplica a lógica do service. Ver os testes obrigatórios em [[Folha de Pagamento]].
