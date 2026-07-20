# Filas e Agendamento (Queues & Scheduler)

## O que é
- **Filas (Queues)**: rodar tarefas **em segundo plano**, fora do ciclo da requisição HTTP (o usuário não espera). Ex.: enviar e-mail, processar um currículo com IA.
- **Agendamento (Task Scheduling)**: rodar tarefas **em horários definidos** (tipo cron). Ex.: fechar a apuração de ponto toda madrugada.

## Por que neste projeto
As decisões de ponto e férias **introduziram tarefas agendadas**:
- **Job diário** que fecha a `PontoApuracaoDiaria` do dia anterior (D-23).
- **Job** que monitora vencimento do período aquisitivo de férias e abre novo (D-27).
E tarefas de fila fazem sentido pra: envio de e-mail (2FA, senha inicial — [[Autenticação e 2FA]]) e a **extração de currículo por IA** (pode demorar, não deve travar a tela — [[Inteligência Artificial]]).

## Fundamentos a dominar
- **Job** — uma classe que representa uma tarefa (`ProcessarApuracaoDiaria`, `ExtrairCurriculo`).
- **Queue worker** — o processo que consome a fila (`php artisan queue:work`).
- **Drivers de fila** — `sync` (roda na hora, dev), `database`, `redis` (produção).
- **Scheduler** — definir tarefas em `routes/console.php`/`Kernel`, disparadas por **um único cron** (`schedule:run` a cada minuto).
- **Idempotência e falhas** — o que acontece se um job falha? Retentativas, jobs "failed", por que a tarefa precisa poder rodar de novo sem duplicar dados.
- **Multitenancy + jobs** — o job precisa saber de qual tenant é (liga com [[Multitenancy (SaaS)]]).

## Onde estudar
- **Laravel docs → "Queues"** e **"Task Scheduling"** — completas e com exemplos.
- **Laravel docs → "Jobs"** (dispatch, retries, failed jobs).
- Laracasts tem séries sobre filas e Redis.

## Como saber que entendeu
Consegue escrever um job que faz a apuração diária de ponto, agendá-lo pra rodar toda madrugada, e explicar o que acontece (e como recuperar) se ele falhar num dia. Ver [[Domínios e Módulos]].
