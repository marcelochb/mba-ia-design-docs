# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** Reunião técnica de webhooks, `[09:03]` a `[09:08]`

## Contexto

A plataforma precisa notificar clientes B2B quando o status de um pedido muda. Hoje não existe nenhum mecanismo de notificação externa, evento ou fila no projeto: os clientes fazem polling em `GET /orders` para descobrir mudanças `[09:00] Marcos`.

A primeira pergunta arquitetural foi se o disparo seria síncrono dentro do service de pedidos ou se passaria por algum mecanismo assíncrono `[09:03] Larissa`.

Dois argumentos eliminaram o disparo síncrono:

1. A transação de mudança de status já é pesada. Ela atualiza `orders`, insere em `order_status_history` e decrementa `stockQuantity` dos produtos do pedido. Acrescentar uma chamada HTTP no meio faria um cliente lento travar a mudança de status de outros pedidos `[09:04] Bruno`.
2. Se o endpoint do cliente estivesse fora do ar, não haveria resposta razoável: dar rollback na mudança de status de um pedido porque a notificação falhou é inaceitável `[09:04] Bruno`.

Ao mesmo tempo, existe um requisito de integridade explícito: não pode existir caso em que o status do pedido muda e o evento não é registrado `[09:40] Bruno`.

Restrição de contexto: o time é pequeno e não quer assumir operação de infraestrutura nova `[09:07] Diego`.

## Decisão

Adotamos o **padrão Outbox persistido no MySQL já existente**.

Quando o status de um pedido muda, **dentro da mesma transação SQL** que atualiza `orders` e insere em `order_status_history`, também é inserida uma linha em uma tabela `webhook_outbox` com o evento `[09:06] Diego`. Um worker separado lê essa tabela e dispara as chamadas HTTP.

A propriedade que isso garante é atomicidade entre a mudança de estado e o registro do evento: se a transação principal commitou, o evento foi registrado; se deu rollback, o evento desaparece junto. Não há janela de inconsistência possível `[09:06] Diego`.

Detalhes de modelagem decididos junto:

- A tabela tem índice no campo de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`. O worker lê apenas os pendentes em batch pequeno, processa e marca como entregue `[09:08] Diego`.
- Chave primária em **UUID**, seguindo o padrão do resto do projeto, onde todos os ids são uuid `[09:51] Larissa`.
- O evento guarda o **payload já renderizado** no momento da inserção (snapshot), não apenas `order_id`. Assim, se o pedido mudar depois, o evento continua refletindo o estado de quando o status mudou `[09:52] Larissa`, `[09:52] Diego`.

## Alternativas Consideradas

### Disparo HTTP síncrono dentro do `changeStatus`

Chamar o endpoint do cliente diretamente na transação de mudança de status.

- **Descartada.** Acopla a latência da mudança de status à disponibilidade do cliente. Um cliente lento degrada a operação de pedidos de todos os outros clientes, e a falha do cliente não tem tratamento possível que preserve a consistência: rollback do status não é opção `[09:04] Bruno`, `[09:06] Diego`.

### Redis Streams ou broker de mensageria dedicado

Publicar o evento em Redis Streams (ou equivalente) e consumir de lá.

- **Descartada.** Exigiria subir infraestrutura nova (Redis Cluster) só para essa feature. Para um time pequeno isso foi classificado como overengineering, e o MySQL existente já resolve o problema `[09:07] Larissa`, `[09:07] Diego`.
- **Trade-off do descarte:** perde-se o throughput e o fan-out nativo de um broker. Em contrapartida, ganha-se atomicidade transacional gratuita (o broker exigiria outbox de qualquer forma para não perder eventos) e zero custo operacional adicional.

### Trigger de banco notificando o worker

Usar trigger no MySQL para avisar o processo consumidor.

- **Descartada.** Tratada em detalhe no [ADR-005](./ADR-005-worker-processo-separado-polling.md): o MySQL não tem listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres, e a trigger só executa SQL, não notifica processo externo `[09:09] Diego`.

## Consequências

### Positivas

- Atomicidade entre mudança de status e registro do evento, sem coordenação distribuída. Atende o requisito de "não pode ter caso de status mudar e evento não sair" `[09:40] Bruno`.
- Nenhuma infraestrutura nova. Reaproveita o MySQL e o Prisma já em produção.
- A outbox funciona como trilha auditável dos eventos gerados, útil para debug e para o histórico de entregas exposto ao cliente.
- O snapshot do payload na inserção elimina a classe de bug em que o evento descreve um estado do pedido diferente daquele que disparou a notificação.

### Negativas

- Latência mínima de entrega limitada pelo intervalo de polling do worker, não pelo tempo da transação. Ver [ADR-005](./ADR-005-worker-processo-separado-polling.md).
- A tabela cresce a cada mudança de status de pedido com webhook interessado. Exige política de arquivamento: linhas entregues são arquivadas depois de ~30 dias, **explicitamente fora do escopo desta feature** `[09:08] Diego`.
- O payload renderizado duplica dados do pedido no banco. Aceito como custo do snapshot; mitigado por manter o payload enxuto, sem os `items` do pedido `[09:43] Diego`.
- A escrita na outbox aumenta marginalmente o tempo da transação de `changeStatus`. Aceito por ser um `INSERT` local, sem I/O de rede.

### Trade-off explícito

Trocamos **throughput e latência mínima** (que um broker dedicado daria) por **simplicidade operacional e atomicidade transacional gratuita**. A troca só se sustenta porque o requisito de latência acordado com os clientes é frouxo: qualquer coisa abaixo de 10 segundos é considerada tempo real `[09:02] Marcos`.

## Decisões relacionadas

- [ADR-005 — Worker em processo separado com polling](./ADR-005-worker-processo-separado-polling.md)
- [ADR-002 — Política de retry com backoff exponencial e DLQ](./ADR-002-retry-backoff-dlq.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./ADR-006-reuso-padroes-existentes.md)
