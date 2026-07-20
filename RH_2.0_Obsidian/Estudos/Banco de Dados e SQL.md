# Banco de Dados e SQL

## O que é
**PostgreSQL** é o banco relacional do projeto (decisão D-03). **SQL** é a linguagem de consulta. Mesmo usando o Eloquent (que gera SQL por você), entender o que acontece embaixo é essencial.

## Por que neste projeto
Toda a persistência (funcionários, folha, ponto) é relacional. Modelagem ruim = bug de dados e folha errada. E, por decisão D-02, **o banco NÃO tem regra de negócio** — só schema e integridade. Entender essa fronteira é chave.

## Fundamentos a dominar
- **Modelo relacional**: tabelas, colunas, tipos, chave primária, **chave estrangeira (FK)**.
- **Relacionamentos**: 1:N (funcionário → holerites), N:N (funcionário ↔ benefícios via tabela pivô).
- **Normalização** básica (por que dependentes viram tabela à parte — ver [[Domínios e Módulos]]).
- **Constraints**: `NOT NULL`, `UNIQUE` (ex.: CPF), `CHECK` (`valor >= 0`). É até onde o banco vai (D-02).
- **SQL essencial**: `SELECT`, `WHERE`, `JOIN`, `GROUP BY`, `ORDER BY` — pra entender o que o Eloquent gera e depurar.
- **Índices** — por que e onde criar (importa pra RNF de uso real, D-15).
- **Transações** (`BEGIN/COMMIT`) e locking — o `ProcessadorFolha` usa transação + `lockForUpdate`.
- Específico do Postgres: tipo **`json`** nativo (usado na extração de currículo).

## Onde estudar
- **PostgreSQL Tutorial** (postgresqltutorial.com) — prático, do básico ao intermediário.
- **SQLBolt** (sqlbolt.com) — exercícios interativos de SQL, rápido.
- **Use The Index, Luke** (use-the-index-luke.com) — como índices funcionam de verdade.
- Documentação oficial do Postgres pra tipos e constraints.

## Como saber que entendeu
Consegue desenhar o schema de `FolhaPagamento` + `LancamentoFolha` com as FKs certas, dizer quais índices criar, e explicar por que o cálculo do IRRF **não** pode ser uma function no banco.
