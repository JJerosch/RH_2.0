# Precisão Decimal e Dinheiro

## O que é
Como representar **valores monetários sem erro de arredondamento**. Em quase toda linguagem, `0.1 + 0.2` **não** dá exatamente `0.3` com `float` (ponto flutuante binário). Numa folha de pagamento, isso é inaceitável. Decisão D-07.

## Por que neste projeto
Cada centavo de INSS, IRRF, FGTS e rescisão precisa bater. Um erro de arredondamento multiplicado por muitos funcionários vira divergência contábil. Por isso o projeto usa **`brick/math`** (`BigDecimal`) em vez de `float`. Ver [[Folha de Pagamento]].

## Fundamentos a dominar
- **Por que `float` falha**: representação binária de ponto flutuante (IEEE 754) não representa exato todo decimal.
- **`brick/math` / `BigDecimal`**: objetos **imutáveis** de precisão arbitrária. Operações (`plus`, `minus`, `multipliedBy`) retornam um novo objeto.
- **Escala e arredondamento**: `toScale(2, RoundingMode::HALF_UP)` — quantas casas e como arredondar. Saber qual modo de arredondamento a folha exige.
- **Nunca misturar** `float` no meio do cálculo — entra `string`/`BigDecimal`, sai `BigDecimal`.
- Como guardar no banco: coluna `decimal(_, 2)`, nunca `float`/`double`.

## Onde estudar
- **brick/math** no GitHub (README + exemplos) — a fonte direta.
- Artigo clássico: **"What Every Computer Scientist Should Know About Floating-Point Arithmetic"** (versão resumida/blog serve).
- Busque "**why not use float for money**" — há ótimas explicações curtas.

## Como saber que entendeu
Consegue explicar por que `CalculadoraIrrf` recebe e devolve `BigDecimal`, o que `toScale(2, HALF_UP)` faz, e por que um `float` ali quebraria a folha.
