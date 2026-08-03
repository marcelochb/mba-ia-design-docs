# ADR-005 — Worker em processo separado consumindo a outbox por polling de 2 segundos

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Marcos (PM)
- **Origem:** Reunião técnica de webhooks, `[09:08]` a `[09:13]`, `[09:29]`

## Contexto

Definido o padrão outbox ([ADR-001](./ADR-001-outbox-no-mysql.md)), restou decidir **como o consumidor lê a tabela** e **onde ele roda** `[09:08] Larissa`.

Duas restrições delimitaram o espaço de solução:

1. **Requisito de latência.** Os clientes consideram tempo real qualquer coisa abaixo de 10 segundos. O que importa para eles é não ficar pendurado atualizando manualmente `[09:02] Marcos`.
2. **Limitação do MySQL.** O MySQL não tem listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres `[09:09] Diego`.

Havia também uma preocupação de disponibilidade: se o worker rodasse dentro da instância da API, um restart da API levaria o worker junto `[09:11] Diego`.

E uma preocupação de performance: se a tabela acumular muitos eventos, o worker fica lento `[09:07] Bruno`.

## Decisão

### Consumo por polling de 2 segundos

O worker roda em loop. A cada **2 segundos**, busca os eventos pendentes mais antigos, processa e marca `[09:09] Diego`.

O intervalo de 2 segundos atende o requisito de "abaixo de 10 segundos" com folga `[09:09] Diego` e foi validado pelo PM `[09:10] Marcos`. A consequência aceita explicitamente: a **latência mínima de entrega passa a ser de 2 segundos no pior caso** `[09:10] Larissa`.

Para o problema de acúmulo, a leitura é feita **em batch pequeno, apenas dos pendentes**, apoiada nos índices de status e `created_at` da outbox `[09:08] Diego`.

### Processo separado da API

O worker **precisa rodar como processo separado**, não dentro da mesma instância da API. Caso contrário, um restart da API derruba o worker `[09:11] Diego`.

A forma decidida acompanha o padrão de entry-point que já existe no projeto: assim como existe `src/server.ts`, será criado um `src/worker.ts`, com um script `npm run worker` `[09:11] Larissa`. A lógica de processamento fica em um arquivo dentro do módulo, como `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` `[09:28] Bruno`.

Mesmo banco e mesma stack, apenas processo distinto `[09:11] Diego`. O worker instancia um **`PrismaClient` próprio**, porque `PrismaClient` é por processo: mesma `DATABASE_URL`, instância nova `[09:30] Bruno`.

### Single-worker e garantia de ordering

Com **um único worker** rodando, os eventos são processados em ordem de `created_at` da outbox, e o cliente recebe em ordem `[09:12] Diego`.

Fica documentado como **limitação conhecida**: não há garantia de ordering global, apenas **por `order_id` e enquanto a operação for single-worker** `[09:13] Larissa`, `[09:12] Diego`.

Isso é aceitável porque os clientes nunca pediram garantia de ordering global. Eles querem saber se cada pedido deles mudou `[09:14] Marcos`.

Escalar para múltiplos workers em paralelo **quebraria essa garantia** `[09:12] Diego`. Os caminhos possíveis quando isso for necessário (particionar por `order_id`, ou usar lock pessimista) foram identificados e **adiados explicitamente**: é problema do futuro, não de agora `[09:13] Diego`.

### Timeout das chamadas HTTP

O worker aplica **timeout de 10 segundos** por chamada. Cliente que não responde em 10s é tratado como falha e marcado para retry `[09:42] Diego`, conforme a política de [ADR-002](./ADR-002-retry-backoff-dlq.md).

## Alternativas Consideradas

### Trigger de banco para consumo reativo

Usar trigger no MySQL para notificar o worker no instante da inserção, em vez de polling `[09:09] Bruno`.

- **Descartada.** O MySQL não tem listener nativo tipo `NOTIFY`/`LISTEN` do Postgres. Trigger existe, mas ela apenas executa SQL: não notifica processo externo. Para avisar o worker seria necessário improvisar algo como escrever em arquivo ou bater em um endpoint, o que foi considerado esquisito `[09:09] Diego`.
- **Trade-off do descarte:** consumo reativo daria latência próxima de zero em vez de até 2 segundos. Perdeu porque o ganho é irrelevante frente ao requisito acordado (abaixo de 10 segundos) e o custo é uma gambiarra de notificação fora do banco.

### Worker dentro do processo da API

Rodar o loop de consumo na mesma instância que serve HTTP.

- **Descartada.** Restart da API derrubaria o worker `[09:11] Diego`. Além disso, acoplaria o ciclo de vida de deploy dos dois componentes e faria o processamento de webhooks competir por event loop com o atendimento de requisições.
- **Trade-off do descarte:** processo único seria mais simples de operar e fazer deploy, com uma só unidade para monitorar. Perdeu para resiliência e independência de ciclo de vida.

### Múltiplos workers em paralelo desde o início

Escalar horizontalmente o consumo já nesta entrega.

- **Descartada / adiada.** Quebraria a garantia de ordering por `order_id`, que é a única garantia de ordenação oferecida `[09:12] Diego`. Exigiria particionamento por `order_id` ou lock pessimista, complexidade classificada como problema do futuro `[09:13] Diego`.
- **Trade-off do descarte:** paralelismo daria throughput maior e resiliência a um worker travado. Perdeu para simplicidade e para a preservação da garantia de ordering, que single-worker entrega de graça.

## Consequências

### Positivas

- Latência de entrega de no máximo ~2 segundos somada ao tempo da chamada HTTP, folgadamente dentro do requisito de 10 segundos `[09:02] Marcos`.
- Ordering por `order_id` garantido sem nenhum mecanismo de coordenação, como efeito colateral gratuito do single-worker.
- Independência de ciclo de vida: deploy ou restart da API não interrompe a entrega de webhooks, e vice-versa.
- Segue um padrão de entry-point que já existe no projeto (`src/server.ts`), sem introduzir conceito estrutural novo `[09:11] Larissa`.
- Nenhuma dependência de recurso de banco não portável. Polling funciona igual em qualquer engine.

### Negativas

- **Single point of failure.** Com um único worker, se ele cai, nenhum evento é entregue até que volte. Não há failover automático nesta entrega. Mitigação: métrica de idade do evento pendente mais antigo, especificada na seção de observabilidade do FDD.
- **Não escala horizontalmente sem quebrar ordering.** O caminho de escala existe, mas está adiado `[09:13] Diego`.
- Polling gera carga constante no banco mesmo sem eventos pendentes: uma query a cada 2 segundos, 24 horas por dia. Mitigado por ser uma query indexada em batch pequeno `[09:08] Diego`.
- Latência mínima de 2 segundos mesmo quando o sistema está ocioso, por construção `[09:10] Larissa`.
- Nova unidade operacional para provisionar, monitorar e fazer deploy, com necessidade de um segundo pool de conexões Prisma `[09:30] Bruno`.

### Trade-off explícito

Trocamos **latência mínima e escalabilidade horizontal** por **simplicidade de implementação e garantia de ordering gratuita**. A troca só é válida sob o requisito de latência frouxo acordado com os clientes; se o requisito apertasse para sub-segundo, esta decisão precisaria ser revisitada.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](./ADR-002-retry-backoff-dlq.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./ADR-006-reuso-padroes-existentes.md)
