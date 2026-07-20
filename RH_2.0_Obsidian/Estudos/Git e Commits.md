# Git e Commits

## O que é
**Git** é o controle de versão: guarda o histórico do código, permite branches e voltar no tempo. O projeto já vive num repositório Git (a pasta `RH_2.0/`).

## Por que neste projeto
O `git log` vai ser seu **mapa de estudo**: cada commit aponta pra decisão que o justifica (ver [[Commits]]). Sem disciplina de Git, você perde o rastro — exatamente o que este cofre existe pra evitar.

## Fundamentos a dominar
- **Conceitos**: repositório, commit, branch, merge, `HEAD`, staging area.
- **Comandos do dia a dia**: `status`, `add`, `commit`, `log`, `diff`, `branch`, `checkout`/`switch`, `merge`, `push`, `pull`.
- **Branch por trabalho**: nunca desenvolver direto no `main` (regra da [[Visão Geral|Definition of Done]]).
- **Conventional Commits**: o padrão `tipo(escopo): resumo` — ver [[Commits]].
- `.gitignore` — por que `.env` e `vendor/` nunca entram no repo (ver [[Segurança e LGPD]]).

## Onde estudar
- **Pro Git** (git-scm.com/book) — gratuito, é *a* referência.
- **Learn Git Branching** (learngitbranching.js.org) — visual e interativo, ótimo pra entender branch/merge.
- Conventional Commits: **conventionalcommits.org**.

## Como saber que entendeu
Consegue criar uma branch, fazer um commit no padrão do projeto referenciando uma nota do cofre, e mesclar no `main` sem medo.
