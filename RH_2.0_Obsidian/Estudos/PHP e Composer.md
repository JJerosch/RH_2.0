# PHP e Composer

## O que é
**PHP** é a linguagem do backend (o Laravel roda sobre ela). **Composer** é o gerenciador de dependências (equivalente ao `npm` do Node ou `pip` do Python) — instala Laravel, brick/math, Filament, etc.

## Por que neste projeto
Tudo é PHP: models, services de cálculo, controllers. Você precisa ler e escrever PHP moderno com conforto pra entender o [[Registro de Decisões|porquê]] e corrigir bugs.

## Fundamentos a dominar
- **Sintaxe básica**: variáveis (`$x`), tipos, arrays associativos, funções.
- **OOP**: classes, objetos, `public/private/protected`, herança, interfaces, `static`, constantes.
- **Tipagem moderna** (PHP 8+): type hints (`int`, `string`, `?Tipo`), `enum`, propriedades tipadas, construtor com promoção (`public function __construct(private Foo $foo)`).
- **Namespaces e autoload** (PSR-4) — como o Composer acha suas classes em `app/Domain/...`.
- **Composer**: `composer.json`, `composer install`, `composer require pacote`, o `vendor/`.
- Recursos usados no plano: `match`, arrow functions (`fn () =>`), null-safe (`?->`).

## Onde estudar
- Documentação oficial: **php.net/manual** (referência).
- **PHP The Right Way** (phptherightway.com) — práticas modernas, curto e direto.
- **Laracasts → "PHP for Beginners"** (série gratuita, excelente).
- Composer: **getcomposer.org/doc**.

## Como saber que entendeu
Consegue ler a classe `CalculadoraIrrf` do plano e explicar o que cada linha faz (o `match`, o type hint `BigDecimal`, o construtor). Ver [[Folha de Pagamento]].
