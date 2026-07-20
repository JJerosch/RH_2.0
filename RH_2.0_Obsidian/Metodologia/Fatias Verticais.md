# Método de Desenvolvimento — Fatias Verticais

O jeito como construímos cada funcionalidade neste projeto. Decisão registrada como D-32 em [[Registro de Decisões]].

## A ideia em uma frase

> Não construímos o sistema em **camadas horizontais** (todo o banco → todo o backend → todo o front). Construímos em **fatias verticais**: uma funcionalidade pequena de cada vez, do banco até a tela, inteira e funcionando.

## Por que (fundamentado no projeto)

- O **valor** do sistema está na **regra de negócio** (cálculo de folha, rescisão, IA), não na tabela nem na tela. Então é por ela que se organiza o trabalho. Ver [[Arquitetura de Software]].
- Construir front primeiro leva à **armadilha do dado falso** — telas com `Math.random()`/mock que parecem funcionar mas não têm nada atrás. **Foi o erro nº 1 e 2 do projeto Java original** (ver [[Folha de Pagamento]]). Fatia vertical proíbe isso: a tela só existe quando o dado real já existe.
- Cada fatia entrega algo **testável e demonstrável**, em vez de meses sem nada rodando.

## A ordem dentro de cada fatia

```
1. Migration + Model      → o dado (guiado pela regra, não o banco inteiro de uma vez)
2. Service + teste (Pest)  → a regra de negócio — o coração, onde mais se aprende
3. Endpoint + feature test → expõe a regra (quando a fatia precisa de API)
4. Tela (Blade ou Filament)→ mostra dado REAL, nunca mock
5. Rota registrada         → só agora, nunca antes
```

Isso é exatamente a **Definition of Done** do [[Plano Técnico (Laravel)]] — uma feature só está "pronta" quando as 5 condições valem juntas.

## Regras práticas

- **Uma fatia = uma funcionalidade pequena e ponta a ponta.** Ex.: "login + 2FA", não "todo o módulo Core".
- **Comece pelo model/migration da fatia**, mas modelado a partir do que a regra precisa — não tente adivinhar o banco inteiro no dia 1.
- **O peso do aprendizado vai no service + teste.** É onde mora o valor e o crescimento como dev.
- **Nunca** faça uma tela com dado falso "só pra ver como fica". Se precisa ver layout, use dado real de um seed pequeno.
- **Trabalho em progresso vive em branch**; só entra no `main`/`master` quando a fatia fecha a DoD. Ver [[Commits]].

## Como isso conversa com o Scrum

Cada **item do Sprint Backlog** ([[Scrum do Projeto]]) é uma fatia vertical. "Terminar o item" = passar pela DoD acima. Assim o método de código e o método de gestão são a mesma coisa vista de dois ângulos.
