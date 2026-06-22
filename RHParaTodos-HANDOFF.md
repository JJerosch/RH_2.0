# Handoff — Reconstrução do RHParaTodos

> Gerado para retomada do trabalho em outra sessão ou por outra pessoa. Cobre toda a conversa, desde a auditoria do projeto original até a troca de stack para Laravel.

---

## 1. Objetivo geral

Redesenhar do zero o projeto **RHParaTodos** (sistema de RH/DP), hoje incompleto em Java/Spring Boot + Thymeleaf + PostgreSQL (`C:\Users\joaoj\Documents\GitHub\RHParaTodos`), como projeto de portfólio. O processo teve 3 etapas:

1. Auditoria do projeto Java original **sem corrigir nada** — só levantar os defeitos reais, lendo o código.
2. Plano técnico de reconstrução em **Django + DRF** (decisão inicial de stack).
3. **Pivô de stack**: decisão final de refazer tudo em **Laravel/PHP**, abandonando Django. A reescrita em Java "para outra hora" também ficou descartada deste ciclo — o foco agora é testar a stack nova.

Diferencial de portfólio definido desde o início: duas features de IA via **Claude API** — explicação de holerite em linguagem natural e extração de dados de currículo — ambas com a regra "IA nunca decide valor monetário, nunca persiste sem revisão humana".

---

## 2. Decisões já tomadas (com o motivo)

| Decisão | Por quê |
|---|---|
| **Nada de trigger/function no banco** — toda regra de negócio (cálculo de INSS/IRRF/FGTS, geração de lançamento, rescisão) vive em código aplicação, testável | No projeto original, ~15 functions PL/pgSQL em `V1__baseline.sql` nunca eram chamadas pela aplicação Java — ficaram mortas. Tabela de IRRF/INSS muda todo ano por lei: precisa estar em código revisável por PR e com teste unitário, não escondido em SQL |
| **Stack final: Laravel + Blade + JS vanilla, sem React** | Pivô explícito do usuário: "quero fazer o projeto de rh apenas com laravel, sem django". Antes disso, a stack travada era Django + DRF + templates Django + JS vanilla — a parte "sem React" se manteve em ambas as versões |
| **PostgreSQL mantido** (não trocado por MySQL, padrão do Laravel) | Mesmo motor do projeto Java original — facilita comparação direta entre as três versões (Spring/Django-planejado/Laravel) |
| **Rescisão com cálculo financeiro completo** (saldo, férias proporcionais, 13º proporcional, aviso prévio, multa FGTS) — não só mudança de status | Usuário escolheu explicitamente "Completo, com cálculo financeiro" quando questionado, sabendo que isso é tão complexo quanto o calculador de INSS/IRRF |
| **2FA recriado, mesma arquitetura do projeto Java, só corrigindo o bug de schema** | A arquitetura (gerar código → e-mail → bloquear rota até validar) já estava certa no Java; o único problema era o hash bcrypt (~60 caracteres) sendo inserido numa coluna `varchar(10)` — custo de refazer corrigido é baixo |
| **PDF de holerite gerado dinamicamente, nunca armazenado** | A fonte de verdade (`LancamentoFolha`) é imutável depois que a folha fecha, então renderizar de novo sempre reproduz o mesmo conteúdo financeiro. Evita custo/complexidade de storage em nuvem sem necessidade real nesse estágio |
| **Prazo: 7-8 meses, sem deadline rígido** | Usuário: "o plano não tem prazo para ser sincero, eu apenas coloquei 6 meses como teto para ficar viável, pode colocar 7 ou 8 meses tranquilo" |
| **README comparativo, seed/fixtures de demo e deploy ficam fora do plano por agora** | Usuário: "Não por agora, mas depois pedirei" |
| **Estrutura de módulos em Laravel: pastas manuais `app/Domain/{Modulo}/`** (não usar pacote `nwidart/laravel-modules`) | Replica a separação por domínio que os "apps" do Django davam, sem adicionar dependência de terceiro (service providers/config por módulo) que não traz benefício real num projeto de um único desenvolvedor |
| **Painel administrativo: Filament** | Equivalente funcional ao Django Admin, que o plano anterior reservava para CRUD interno de RH/DP (`organizacao`, `funcionarios`, `ponto`, `ferias`, `beneficios`, `solicitacoes`, `relatorios`). Blade+JS customizado fica só para telas voltadas a funcionário/candidato (`meus-holerites`, Kanban de recrutamento) |
| **Cliente de IA: wrapper próprio sobre Guzzle, REST direto** (não pacote de comunidade) | Não existe SDK oficial da Anthropic em PHP (só Python/TypeScript/Java/Go/Ruby) — wrapper próprio dá mais controle que depender de pacote não-oficial e potencialmente mal mantido |
| **PDF (holerite, perfil de candidato): `spatie/laravel-pdf` (Browsershot/Chromium)**, não `barryvdh/laravel-dompdf` | CSS completo, equivalente em fidelidade ao `WeasyPrint` do plano Django. `dompdf` tem suporte limitado a CSS moderno (sem flexbox), o que limitaria o design dos templates |
| **Precisão decimal: `brick/math` (`BigDecimal`)**, não `float` nem `bcmath` puro | PHP não tem `Decimal` nativo como o Python. `float` em cálculo de folha é inaceitável (erro de arredondamento). `brick/math` dá objetos imutáveis de precisão arbitrária — papel equivalente ao `decimal.Decimal` |
| **Estimativa total: ~280-300h** (vs. ~265-273h no plano Django) | Aumento concentrado na curva de aprendizado do Filament (Fase 1) e na ausência de SDK oficial de IA em PHP (Fase 5) — a lógica de negócio em si é idêntica |

---

## 3. Decisões descartadas ou revertidas

| O que foi descartado | Por quê |
|---|---|
| **Django + DRF como framework principal** | Pivô explícito do usuário pra Laravel. Todo o plano em Django (`RHParaTodos-Django-plano.md`) foi **mantido como arquivo histórico**, não apagado, mas não é mais o plano corrente |
| **React no frontend** | Rejeitado desde a primeira versão do plano ("sem React por agora") — decisão carregada sem mudança para o plano Laravel |
| **`nwidart/laravel-modules`** (pacote de módulos do Laravel) | Considerado e descartado em favor de pastas manuais `app/Domain/` — citado explicitamente no plano Laravel como decisão tomada, não como ponto aberto |
| **Rescisão básica (só mudar status, sem cálculo financeiro)** | Era a opção mais simples/rápida, mas o usuário escolheu a completa de propósito, mesmo sabendo do custo extra de horas |
| **Reescrever o projeto em Java imediatamente** | Usuário: a versão Java fica "para outra hora" — o ciclo atual é só sobre testar a stack nova (era Django, agora é Laravel) |
| **Abandonar o 2FA por ser complexo** | Usuário pediu explicitamente pra recriar, não cortar — perguntou "você pretende fazer como?" em vez de aceitar tirar do escopo |
| **`barryvdh/laravel-dompdf` como biblioteca de PDF** | Considerado, descartado em favor de `spatie/laravel-pdf` por limitação de CSS |

---

## 4. Arquivos criados ou alterados

| Arquivo | Caminho | Estado | Conteúdo |
|---|---|---|---|
| `RHParaTodos-Django-plano.md` | `C:\Users\joaoj\Documents\GitHub\RHParaTodos-Django-plano.md` | **Intacto, não editado desde a criação** — mantido como histórico da decisão anterior | 417 linhas. Plano técnico completo em Django: contexto/auditoria (6 falhas originais), decisão de arquitetura (sem trigger), stack Django, estrutura de apps, app `core` (2FA), app `folha` (calculadoras INSS/IRRF/FGTS, `processador_folha`, rescisão completa com `regras_por_tipo_desligamento`), app `ia` (explicador de holerite + extrator de currículo), tabela de correções, cronograma de 7-8 meses (~265-273h), Definition of Done |
| `RHParaTodos-Laravel-plano.md` | `C:\Users\joaoj\Documents\GitHub\RHParaTodos-Laravel-plano.md` | **Criado, é o plano corrente** | Reescrita completa do plano Django em termos Laravel/PHP/Eloquent/Blade/Filament. Mesma arquitetura de negócio, mesmas regras de rescisão e segurança de IA, só troca de implementação técnica. Inclui tabela comparativa de stack, decisão de estrutura de módulos, decisão do Filament, cronograma revisado (~280-300h) |
| `DEVELOPMENT_GUIDE.md` | `C:\Users\joaoj\Documents\GitHub\DEVELOPMENT_GUIDE.md` | **⚠️ Desatualizado — ainda em termos Django, NÃO foi reescrito para Laravel** | 554 linhas, 15 seções (Visão Geral, Escopo do MVP, Personas, Requisitos Funcionais RF-001 a RF-058, Requisitos Não Funcionais RNF-001 a RNF-008, Casos de Uso por módulo, Fluxos de Negócio, Estrutura de Dados, Contratos de API, Estrutura de Frontend, Estratégia de Testes, Roadmap Técnico, Critérios de Aceite, Checklist de Desenvolvimento, Riscos Técnicos). Todo grounded estritamente no `RHParaTodos-Django-plano.md`, com gaps marcados como "⚠️ Lacuna no plano". **Uma tentativa de atualização foi iniciada (comando do usuário "refaa") mas foi interrompida antes de qualquer edição ser feita** — o arquivo no disco continua 100% em termos Django |
| `sequential-foraging-cherny.md` | `C:\Users\joaoj\.claude\plans\sequential-foraging-cherny.md` | Scratch file de Plan Mode, não é deliverable | Usado uma vez pra estagiar os achados de rescisão/2FA/ponto-férias antes de uma rodada de perguntas ao usuário. Não precisa ser mantido |
| `plano-fullstack.html` | `C:\Users\joaoj\Documents\GitHub\Faculdade\plano-fullstack.html` | **Não editado — leitura apenas, e agora desatualizado** | Plano de estudos PACER do usuário. Fase 2 (ago-set/2026) cita "RHParaTodos versão **Django**" como entregável — ⚠️ ficou inconsistente com o pivô para Laravel, e ninguém pediu correção disso ainda |

---

## 5. Pontos em aberto (perguntas exatas, já adaptadas para Laravel)

### Bloqueante pra começar (Fase 0-1)
1. **Perfis de acesso**: qual é a lista fechada de perfis (`ADMIN`, `DP_CHEFE`, "RH/DP" — mesmo perfil ou distintos)? Existe perfil "Gestor" (citado na estrutura pedida pro guia, mas nunca no plano)? Candidato tem login, ou fluxo é público até confirmação do RH?
2. **2FA**: quantas tentativas (N) até bloquear? Tempo de expiração exato do código (plano só dá "ex.: 5 min")? Existe reenvio de código, ou só gerar novo? O que acontece após bloqueio — espera automática ou desbloqueio manual?
3. **Funcionário (cadastro manual via Filament)**: quais campos são obrigatórios? Fluxo de promoção — quem aprova, o que muda exatamente?
4. **Benefícios**: que campos tem `TipoBeneficio` (valor, recorrência, desconto automático)? Como `FuncionarioBeneficio` entra no `GeradorLancamentos` — automático ou manual?

### Importante antes da Fase 1-2
5. **Ponto**: como uma marcação (`PontoMarcacao`) se transforma em apuração diária (`PontoApuracaoDiaria`)? Existe regra de tolerância/jornada? `PontoOcorrencia`/`PontoCalendario` servem pra quê?
6. **Férias**: fluxo de solicitação (`FeriasSolicitacao`) — quem aprova, quantas etapas, passa por Filament ou tela própria? Como o período aquisitivo é atualizado com o tempo?

### Importante antes da Fase 3
7. **Recrutamento ponta a ponta**: quais etapas exatas o Kanban percorre? O que dispara a criação do `Funcionario` (mudança de etapa específica)? O que acontece se o CPF do candidato já existir como funcionário? A candidatura é criada automaticamente após o RH confirmar a extração por IA, ou precisa de ação extra?

### Decide o tamanho do trabalho, mas pode esperar
8. **Folha**: nome do tipo "mensal" padrão de `FolhaPagamento` (só `RESCISAO` está nomeado)? Reabrir folha sem ser ADMIN/DP_CHEFE — 403 simples ou mensagem específica? Explicação por IA só em folha fechada, ou também aberta?
9. **Contratos de API**: payload/resposta de cada um dos 6 endpoints de folha — definir agora ou endpoint por endpoint? Módulos administrativos ficam só em Filament (sem API REST própria), ou precisam de endpoint JSON pra consumo externo/mobile futuro?
10. **Frontend**: lista completa de telas Blade fora do Filament — reaproveitar layout visual do Spring original ou desenhar do zero? Confirma que Filament cobre todo CRUD administrativo e só `meus-holerites`/Kanban ficam em Blade customizado?
11. **Relatórios**: formato de saída (tela Filament, PDF, CSV)? Quais filtros? Lista fechada definitiva de quais relatórios entram (hoje só 3 exemplos: headcount, custo de folha, férias vencendo)?
12. **App `solicitacoes`**: o que é esse módulo exatamente — substitui os fluxos específicos (férias/promoção/desligamento) ou coexiste com eles?

### Decisão de escopo, não de detalhe
13. **Auditoria/LGPD**: vale ter log de alterações nesse estágio de portfólio (o Spring original tinha trigger pra isso, plano novo não propõe substituto)? Em Laravel seria via `spatie/laravel-activitylog` ou Observer próprio.
14. **Performance/Disponibilidade/Escalabilidade**: sistema é só demo local, ou há expectativa real de uso? Decide se vale meta de RNF ou se fica formalmente fora de escopo.
15. **Testes E2E**: vale investir em E2E (Playwright ou Dusk nativo do Laravel) dado o orçamento já maior (~280-300h), ou ficar só com unit + feature test no MVP?

### Específicos da troca de stack
16. **Ambiente de PDF**: `spatie/laravel-pdf` depende de Node + Chromium (via Browsershot) — viável no setup local/deploy do usuário, ou cair pra `dompdf` (mais leve, CSS limitado)?
17. **Confiabilidade do cliente de IA sem SDK oficial**: como o wrapper Guzzle vai tratar retry/timeout/rate limit? Escrever essa camada já na Fase 5, ou aceitar versão mínima sem retry no MVP?

---

## 6. Próximo passo concreto

Duas ações estão pendentes e **nenhuma das duas foi confirmada pelo usuário** — a sessão foi interrompida no meio da primeira:

1. **Reescrever `DEVELOPMENT_GUIDE.md` para Laravel.** Foi sinalizado duas vezes como desatualizado (ainda fala em Django/DRF/Eloquent não, etc.) e uma tentativa de reescrita foi iniciada via comando do usuário ("refaa") mas **interrompida antes de qualquer edição acontecer** — o arquivo no disco não mudou. Isso é o item mais imediato e mais barato de resolver primeiro.
2. **Decidir as 17 perguntas da seção 5** — sem isso, várias seções do `DEVELOPMENT_GUIDE.md` (quando reescrito) continuarão cheias de "⚠️ Lacuna no plano".

**Recomendação de ordem**: primeiro reescrever o `DEVELOPMENT_GUIDE.md` espelhando o `RHParaTodos-Laravel-plano.md` (mesmo nível de detalhe e mesmo sistema de marcação de lacunas usado na versão Django, só que agora as lacunas reais são as 17 perguntas listadas acima) — depois, conforme o usuário for respondendo as perguntas, atualizar ambos os documentos incrementalmente.

---

## 7. Contexto implícito (convenções e preferências informais)

- **Idioma de trabalho**: toda a comunicação com o usuário é em português (Brasil).
- **Nunca editar um documento de plano sem instrução explícita.** Quando a stack mudou de Django pra Laravel, a resposta certa foi criar um arquivo **novo** (`RHParaTodos-Laravel-plano.md`) e deixar o antigo intacto — não sobrescrever histórico de decisão.
- **Honestidade sobre lacunas é uma exigência de estilo, não só de conteúdo.** O usuário pediu explicitamente "extremamente detalhado" com toda lacuna flagada como "⚠️ Lacuna no plano" — não preencher buraco com suposição razoável sem marcar.
- **"Pode mudar" / aprovações genéricas autorizam decisões de implementação a meu critério, desde que documentadas com o porquê** — não é necessário voltar a perguntar item por item quando o usuário já deu sinal verde geral (foi assim que as decisões de Filament, `spatie/laravel-pdf`, `brick/math` e cliente Guzyle foram tomadas, todas justificadas no próprio plano).
- **A auditoria do projeto Java original é canônica e não deve ser re-derivada.** Os 6 problemas originais + 4 adicionais (rescisão sem cálculo, bug de schema do 2FA, falta de rastreabilidade vaga→funcionário, ponto/férias desconectados da folha) já foram confirmados por leitura direta de código (`V1__baseline.sql`, `TwoFactorCodeStore.java`, `SolicitacaoService.java`, `Vaga.java`, `Candidatura.java`, etc.) — não precisam ser auditados de novo.
- **Arquivos de plano/guia ficam direto em `C:\Users\joaoj\Documents\GitHub\`**, não dentro do repositório `RHParaTodos` em si — são documentos de planejamento paralelos ao código.
- **Pedidos de "lista de perguntas" ou "resumo" são respostas em texto/chat por padrão**, só vão para arquivo `.md` quando o usuário pede explicitamente (como aconteceu com este próprio handoff).
