# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em arquivo individual, no formato
`ADR-NNN-titulo-em-kebab-case.md`, seguindo o padrão MADR.

## Índice

| ADR | Decisão | Status | Origem na transcrição |
| --- | --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos | Aceita | `[09:03]` a `[09:08]` |
| [ADR-002](./ADR-002-retry-backoff-dlq.md) | Retry com backoff exponencial de 5 tentativas e DLQ em tabela separada | Aceita | `[09:14]` a `[09:19]` |
| [ADR-003](./ADR-003-at-least-once-x-event-id.md) | Garantia at-least-once com deduplicação por `X-Event-Id` no cliente | Aceita | `[09:24]` a `[09:26]` |
| [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period | Aceita | `[09:19]` a `[09:23]`, `[09:36]` |
| [ADR-005](./ADR-005-worker-processo-separado-polling.md) | Worker em processo separado consumindo a outbox por polling de 2 segundos | Aceita | `[09:08]` a `[09:13]`, `[09:29]` |
| [ADR-006](./ADR-006-reuso-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto no módulo de webhooks | Aceita | `[09:27]` a `[09:30]` |

O conjunto cobre as 6 decisões principais discutidas na reunião. O ADR-006
referencia explicitamente arquivos e classes do código base existente.

## Decisões secundárias não promovidas a ADR

Os itens abaixo foram decididos na reunião, mas classificados como detalhe de
implementação ou requisito não funcional, e vivem apenas no [FDD](../FDD.md):

| Item | Onde ficou | Justificativa da classificação |
| --- | --- | --- |
| TLS obrigatório na URL do webhook (`https`) | FDD, contratos e matriz de erros | Classificado na reunião como validação de schema Zod, não decisão arquitetural `[09:23] Sofia` |
| Limite de 64KB de payload, com erro ao ultrapassar | FDD, requisitos e matriz de erros | Classificado na reunião como requisito não funcional, não decisão arquitetural separada `[09:24] Larissa` |
| Timeout de 10s na chamada HTTP do worker | FDD, resiliência; citado em ADR-005 | Parâmetro de configuração da política de entrega `[09:42] Diego` |
| Formato do payload do evento | FDD, contratos | Detalhe de contrato `[09:43] Diego` |
| Headers de envio (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) | FDD, contratos; `X-Event-Id` em ADR-003 | Detalhe de contrato `[09:44] Diego`, `[09:44] Sofia` |
| Filtro de eventos aplicado na inserção da outbox | FDD, fluxos | Detalhe de implementação `[09:34] Bruno` |
| Snapshot do payload renderizado na inserção | FDD, fluxos; citado em ADR-001 | Detalhe de modelagem `[09:52] Larissa` |
