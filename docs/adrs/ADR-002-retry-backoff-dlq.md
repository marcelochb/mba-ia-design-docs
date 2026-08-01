# ADR-002 — Retry com backoff exponencial de 5 tentativas e DLQ em tabela separada

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** Reunião técnica de webhooks, `[09:14]` a `[09:19]`

## Contexto

Endpoints de webhook de clientes ficam indisponíveis. A pergunta levantada foi direta: se o cliente está offline no momento do envio, o que o sistema faz `[09:14] Larissa`.

Sem política de retry, um evento perdido durante uma janela de indisponibilidade do cliente nunca chega, o que derrota o propósito da feature: os clientes pediram webhooks justamente para parar de fazer polling `[09:00] Marcos`.

O dado concreto que calibrou a decisão: já houve cliente da plataforma com indisponibilidade de duas horas em manutenção planejada `[09:16] Diego`.

Ao mesmo tempo, retry não pode ser infinito. Retry indefinido com backoff traz o problema de o evento ficar pendurado para sempre quando o cliente simplesmente desapareceu `[09:15] Diego`.

## Decisão

Adotamos **backoff exponencial com 5 tentativas** e, no esgotamento, movimentação do evento para uma **Dead Letter Queue persistida em tabela separada**.

**Progressão do backoff:** 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas `[09:17] Diego`.

Isso dá uma janela de quase 15 horas entre a primeira falha e a última tentativa `[09:17] Diego`. O critério de aceitação de negócio foi explícito: um cliente que fica fora por 15 horas já tem um problema sério do lado dele `[09:17] Marcos`.

**DLQ em tabela separada** (`webhook_dead_letter`), guardando o payload, o motivo da falha e o timestamp `[09:18] Diego`. A escolha por tabela separada em vez de um status `failed` na própria outbox tem duas razões:

1. Mantém a leitura da outbox principal limpa, já que o worker varre apenas os pendentes.
2. A DLQ funciona como evidência para debug e base para reprocessamento `[09:18] Diego`.

**Reprocessamento é manual, via endpoint administrativo:** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente `[09:18] Diego`, `[09:35] Diego`. O controle de acesso desse endpoint está em [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md#autorização-do-endpoint-administrativo) e detalhado no FDD.

## Alternativas Consideradas

### 3 tentativas com backoff mais agressivo

Proposta de reduzir para 3 tentativas, por ser mais agressivo no descarte `[09:16] Bruno`.

- **Descartada.** Três tentativas cobririam uma janela de aproximadamente 30 minutos. Um cliente com indisponibilidade matinal teria seus eventos mortos antes de voltar. O caso real de manutenção planejada de duas horas não seria coberto `[09:16] Diego`.
- **Trade-off do descarte:** 3 tentativas liberariam a linha da outbox mais rápido e reduziriam o volume de retentativas. Perdeu para a cobertura de janelas reais de indisponibilidade.

### Retry indefinido com backoff

Nunca desistir, apenas espaçar cada vez mais as tentativas.

- **Descartada.** Evento fica pendurado para sempre se o cliente sumiu, e a outbox nunca converge. Sem teto, não existe momento em que o operador é notificado de que a entrega falhou de forma definitiva `[09:15] Diego`.
- **Trade-off do descarte:** retry indefinido maximizaria a chance de entrega eventual. Perdeu para a previsibilidade operacional e para a necessidade de um sinal claro de falha permanente.

### Status `failed` na própria tabela de outbox, sem DLQ separada

Marcar o evento como falho na outbox em vez de movê-lo `[09:17] Larissa`.

- **Descartada.** Poluiria a leitura da outbox principal e misturaria o ciclo de vida de entrega ativa com o de falhas permanentes `[09:18] Diego`.
- **Trade-off do descarte:** uma tabela só seria mais simples de modelar e evitaria a movimentação de linha. Perdeu para clareza de leitura e para o uso da DLQ como evidência de debug.

## Consequências

### Positivas

- Cobre janelas reais de indisponibilidade de cliente, incluindo o caso concreto de duas horas de manutenção planejada `[09:16] Diego`.
- Teto de tentativas garante convergência: todo evento termina como entregue ou na DLQ. Não existe estado pendurado indefinidamente.
- A DLQ dá visibilidade operacional sobre falhas permanentes e viabiliza reprocessamento sem intervenção manual no banco.
- Backoff exponencial evita martelar um endpoint que já está em dificuldade, comportamento de bom cidadão em relação à infraestrutura do cliente.

### Negativas

- Latência de entrega em caso de falha pode chegar a ~15 horas. Aceito explicitamente pelo PM `[09:17] Marcos`.
- Reprocessamento manual exige ação humana. Não há re-drive automático da DLQ, o que significa que eventos na DLQ ficam parados até alguém agir.
- Não há notificação proativa ao cliente quando o webhook dele começa a falhar. Foi pedido e **descartado desta fase**: aviso por email fica para a próxima fase, depois de medir o impacto `[09:37] Marcos`, `[09:37] Larissa`.
- Um evento com 5 tentativas mantém a linha ocupada na outbox por até 15 horas, competindo com a leitura dos pendentes. Mitigado pelo batch pequeno e pelo índice em status.

### Trade-off explícito

Trocamos **tempo de convergência** por **cobertura de indisponibilidade**. Uma janela de 15 horas é longa para um sistema de notificação, mas o alternativo (matar em 30 minutos) falharia em casos de indisponibilidade que já aconteceram de fato na base de clientes.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./ADR-001-outbox-no-mysql.md)
- [ADR-003 — Entrega at-least-once com X-Event-Id](./ADR-003-at-least-once-x-event-id.md)
- [ADR-005 — Worker em processo separado com polling](./ADR-005-worker-processo-separado-polling.md)
