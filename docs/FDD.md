# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Versão** | 1.0 |
| **Data** | 2026-08-01 |
| **Responsável técnico** | Larissa (Tech Lead) |
| **Revisores técnicos** | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |
| **Origem** | Reunião técnica de webhooks (`TRANSCRICAO.md`) e código base do OMS |
| **Documentos relacionados** | [PRD](./PRD.md), [RFC](./RFC.md), [ADRs](./adrs/), [Tracker](./TRACKER.md) |

> Este documento especifica **como construir**. A justificativa de cada decisão está nos [ADRs](./adrs/); a proposta em nível de arquitetura está no [RFC](./RFC.md).

---

## 1. Contexto e motivação técnica

O OMS não possui nenhum mecanismo de notificação externa, evento, fila ou webhook. A mudança de status de um pedido acontece hoje em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`), dentro de uma transação Prisma que atualiza `orders`, insere em `order_status_history` e ajusta `stockQuantity` dos produtos. Nada disso é observável de fora da aplicação.

O problema técnico a resolver: **publicar mudanças de status para endpoints HTTP de terceiros sem acoplar a disponibilidade da operação de pedidos à disponibilidade desses endpoints**, e sem abrir janela de inconsistência entre "status mudou" e "evento publicado" `[09:40] Bruno`.

**Atores:**

| Ator | Papel |
| --- | --- |
| API OMS (`src/server.ts`) | Recebe `PATCH /orders/:id/status` e grava o evento na outbox dentro da transação |
| Worker (`src/worker.ts`) | Processo separado que consome a outbox e entrega os eventos |
| Endpoint do cliente | Recebe `POST` HTTPS assinado com HMAC. Fora da nossa infraestrutura |
| Operador ADMIN | Reprocessa eventos da DLQ via endpoint administrativo |

**Limites do escopo técnico:** apenas webhooks **outbound**. A plataforma não recebe webhooks de clientes `[09:02] Marcos`, `[09:03] Sofia`.

**Suposições e restrições:**

- Latência de entrega aceitável abaixo de 10 segundos `[09:02] Marcos`, o que viabiliza polling de 2s.
- MySQL não oferece `NOTIFY`/`LISTEN` `[09:09] Diego`, o que elimina consumo reativo por trigger.
- Time pequeno, sem apetite para infraestrutura nova `[09:07] Diego`.
- Um único worker em execução. Ordering garantido apenas por `order_id` sob essa premissa `[09:13] Larissa`.

---

## 2. Objetivos técnicos

| Objetivo | Medida ou invariante |
| --- | --- |
| Atomicidade entre mudança de status e registro do evento | **Invariante:** se `orders.status` foi alterado e existe webhook ativo interessado no status destino, existe linha correspondente em `webhook_outbox`. Se a transação falhar, nem o status nem o evento persistem `[09:40] Bruno`, `[09:41] Diego` |
| Entrega dentro do requisito de latência | p95 do tempo entre commit da transação e primeira tentativa de entrega ≤ 5s; teto de polling em 2s `[09:09] Diego` |
| Nenhum evento perdido nem pendurado indefinidamente | **Invariante:** todo evento termina em estado `DELIVERED` ou movido para `webhook_dead_letter`. Não há estado terminal ambíguo `[09:15] Diego` |
| Ordering por pedido | **Invariante:** para um mesmo `order_id`, os eventos são entregues em ordem crescente de `created_at`, enquanto houver um único worker `[09:12] Diego` |
| Autenticidade e integridade verificáveis pelo cliente | Toda requisição sai com `X-Signature` = HMAC-SHA256 do corpo, com secret exclusiva do endpoint `[09:20] Sofia`, `[09:21] Sofia` |
| Nenhuma degradação do fluxo de pedidos existente | A operação acrescentada a `changeStatus` é um `INSERT` local, sem I/O de rede dentro da transação |
| Zero alteração no tratamento de erro existente | Erros do módulo estendem `AppError` e são serializados pelo `errorMiddleware` atual sem modificação `[09:29] Bruno` |

---

## 3. Escopo e exclusões

### Incluído

- Modelagem de `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` em `prisma/schema.prisma`.
- Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas `[09:27] Bruno`.
- Gancho transacional em `OrderService.changeStatus` via função `publishWebhookEvent(tx, ...)` `[09:41] Bruno`.
- Worker em processo separado (`src/worker.ts` + `webhook.worker.ts`) com polling de 2s `[09:11] Larissa`, `[09:28] Bruno`.
- Assinatura HMAC-SHA256, geração e rotação de secret com grace period de 24h `[09:22] Sofia`.
- Retry com backoff 1m/5m/30m/2h/12h e DLQ `[09:17] Diego`.
- CRUD de configuração de webhook, histórico de entregas e endpoint admin de replay.
- Filtro de eventos por status na inserção da outbox `[09:34] Bruno`.

### Excluído

- **Notificação por email** ao cliente quando o webhook falha. Adiado para a próxima fase `[09:37] Larissa`.
- **Dashboard visual** do cliente. Projeto separado do time de frontend `[09:40] Larissa`.
- **Rate limiting de saída** por cliente. Observar e decidir depois `[09:39] Diego`.
- **Arquivamento** de linhas entregues da outbox. Fora do escopo desta feature `[09:08] Diego`.
- **Webhooks inbound** `[09:02] Marcos`.
- **Múltiplos workers em paralelo** e particionamento por `order_id` `[09:13] Diego`.
- **Exactly-once**. A garantia é at-least-once `[09:24] Diego`.

---

## 4. Modelagem de dados

Todas as tabelas seguem as convenções já vigentes em `prisma/schema.prisma`: `id` em `String @id @default(uuid()) @db.Char(36)`, `@@map` explícito e `@@index` nos campos de consulta `[09:51] Larissa`.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model WebhookEndpoint {
  id            String   @id @default(uuid()) @db.Char(36)
  customerId    String   @db.Char(36)
  url           String   @db.VarChar(500)
  secret        String   @db.VarChar(255)
  previousSecret         String?   @db.VarChar(255)
  previousSecretExpiresAt DateTime?
  subscribedStatuses    Json
  active        Boolean  @default(true)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id           String              @id @default(uuid()) @db.Char(36)
  eventType    String              @db.VarChar(64)
  orderId      String              @db.Char(36)
  webhookEndpointId String         @db.Char(36)
  payload      Json
  status       WebhookOutboxStatus @default(PENDING)
  attempts     Int                 @default(0)
  nextAttemptAt DateTime           @default(now())
  lastError    String?             @db.VarChar(500)
  createdAt    DateTime            @default(now())
  updatedAt    DateTime            @updatedAt

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id                String   @id @default(uuid()) @db.Char(36)
  webhookEndpointId String   @db.Char(36)
  outboxEventId     String   @db.Char(36)
  attempt           Int
  success           Boolean
  responseStatus    Int?
  responseBody      String?  @db.Text
  durationMs        Int
  errorMessage      String?  @db.VarChar(500)
  sentAt            DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([webhookEndpointId, sentAt])
  @@index([outboxEventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id                String   @id @default(uuid()) @db.Char(36)
  outboxEventId     String   @db.Char(36)
  webhookEndpointId String   @db.Char(36)
  eventType         String   @db.VarChar(64)
  orderId           String   @db.Char(36)
  payload           Json
  failureReason     String   @db.VarChar(500)
  totalAttempts     Int
  replayedAt        DateTime?
  replayedById      String?  @db.Char(36)
  createdAt         DateTime @default(now())

  @@index([webhookEndpointId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}
```

**Notas de modelagem:**

- `payload` é `Json` e guarda o **snapshot renderizado na inserção**. Se o pedido mudar depois, o evento continua refletindo o estado do momento da mudança de status `[09:52] Larissa`, `[09:52] Diego`.
- Índice composto `[status, nextAttemptAt]` sustenta a query de polling do worker `[09:08] Diego`.
- `previousSecret` e `previousSecretExpiresAt` implementam o grace period de 24h `[09:21] Sofia`.
- `subscribedStatuses` é a lista de status que aquele endpoint quer ouvir `[09:33] Marcos`.
- Uma mudança de status gera **uma linha de outbox por endpoint interessado**, o que mantém `attempts` e `nextAttemptAt` independentes por destino.

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro da transação de `changeStatus`)

Ponto de integração: `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`).

1. O cliente chama `PATCH /api/v1/orders/:id/status`, validado por `updateOrderStatusSchema` (`src/modules/orders/order.schemas.ts`).
2. `changeStatus` abre `this.prisma.$transaction`.
3. Fluxo existente, inalterado: carrega o pedido, valida `from === to` (`ConflictError`), valida `canTransition(from, to)` (`InvalidStatusTransitionError`), aplica `debitStock`/`replenishStock` conforme `shouldDebitStock`/`shouldReplenishStock`, atualiza `orders.status` e insere em `order_status_history`.
4. **Novo passo, ainda dentro do mesmo `tx`:** chamada a `publishWebhookEvent(tx, order, from, to)` `[09:41] Bruno`.
5. Dentro de `publishWebhookEvent`:
   1. Consulta `webhook_endpoints` do `order.customerId` com `active = true`.
   2. **Filtra os endpoints cujo `subscribedStatuses` contém o status destino.** Se nenhum endpoint quer aquele status, **retorna sem inserir nada**, economizando linha na tabela `[09:34] Bruno`, `[09:34] Diego`.
   3. Renderiza o payload do evento (snapshot, seção 6.5).
   4. Valida o tamanho serializado do payload contra o teto de **64KB**. Se ultrapassar, lança `WebhookPayloadTooLargeError` `[09:24] Diego`, `[09:24] Larissa`.
   5. Insere uma linha em `webhook_outbox` por endpoint, com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
6. A transação commita. **Se o `INSERT` da outbox falhar, a transação inteira faz rollback**, incluindo a mudança de status e o ajuste de estoque `[09:40] Bruno`. Esse é o comportamento desejado: não pode existir caso de status mudar e evento não sair `[09:41] Diego`.

**Assinatura proposta** `[09:41] Bruno`, `[09:41] Diego`:

```ts
// src/modules/webhooks/webhook.publisher.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus | null,
  toStatus: OrderStatus,
): Promise<void>;
```

Função que recebe o `tx` client da transação corrente, sem injetar o repository inteiro no `OrderService` `[09:41] Diego`.

### 5.2 Processamento pelo worker

Entry-point: `src/worker.ts`. Lógica: `src/modules/webhooks/webhook.worker.ts` `[09:28] Bruno`.

1. Bootstrap: instancia `PrismaClient` próprio via `createPrismaClient()` (`src/config/database.ts`), porque `PrismaClient` é por processo `[09:30] Bruno`. Mesma `DATABASE_URL`.
2. Loop com intervalo de **2 segundos** `[09:09] Diego`.
3. A cada ciclo, busca em batch pequeno (default `WEBHOOK_WORKER_BATCH_SIZE = 20`) os eventos com `status = PENDING` e `nextAttemptAt <= now()`, ordenados por `created_at` crescente `[09:08] Diego`, `[09:12] Diego`.
4. Para cada evento, sequencialmente (single-worker preserva ordering por `order_id` `[09:12] Diego`):
   1. Marca `status = PROCESSING`.
   2. Carrega o endpoint de destino. Se estiver inativo ou removido, encerra o evento sem tentativa de rede e registra o motivo.
   3. Calcula `X-Signature` = HMAC-SHA256 do corpo serializado, com a secret vigente do endpoint `[09:20] Sofia`.
   4. Faz `POST` na URL do endpoint com os headers da seção 6.4 e **timeout de 10 segundos** `[09:42] Diego`.
   5. Registra o resultado em `webhook_deliveries`: `attempt`, `success`, `responseStatus`, `responseBody` (truncado), `durationMs`.
   6. **Sucesso** (resposta `2xx`): `status = DELIVERED`.
   7. **Falha** (timeout, erro de rede, resposta não-`2xx`): incrementa `attempts` e aplica a política de retry da seção 5.3.
5. Shutdown gracioso em `SIGINT`/`SIGTERM`, encerrando o ciclo corrente antes de sair e chamando `prisma.$disconnect()`, seguindo o padrão de `src/server.ts`.

### 5.3 Retry com backoff

Política: **5 tentativas**, backoff **1m, 5m, 30m, 2h, 12h** `[09:17] Diego`.

| Tentativa concluída com falha | `nextAttemptAt` | Acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 | `now() + 1 minuto` | 1m |
| 2 | `now() + 5 minutos` | ~6m |
| 3 | `now() + 30 minutos` | ~36m |
| 4 | `now() + 2 horas` | ~2h36m |
| 5 | — move para DLQ | ~14h36m |

1. Em falha, incrementa `attempts` e grava `lastError` truncado em 500 caracteres.
2. Se `attempts < 5`: `status = PENDING` e `nextAttemptAt = now() + backoff[attempts]`. O evento volta a ser elegível no ciclo de polling correspondente.
3. Se `attempts >= 5`: aplica o fluxo de DLQ (5.4).

O `X-Event-Id` **permanece o mesmo em todas as tentativas**, o que é o que viabiliza a deduplicação no cliente `[09:25] Diego`.

### 5.4 Dead Letter Queue e replay

**Movimentação para a DLQ**, após a 5ª falha `[09:18] Diego`:

1. Em transação: insere em `webhook_dead_letter` com `payload`, `failureReason`, `totalAttempts` e `outboxEventId`, e marca a linha da outbox como `FAILED`.
2. Incrementa a métrica `webhook_dead_letter_total` e emite log em nível `error`.

**Replay manual**, via `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN` `[09:35] Diego`, `[09:36] Sofia`:

1. `authenticate` + `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`) `[09:36] Larissa`.
2. Carrega a linha da DLQ. Se não existir, `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
3. Se `replayedAt` já estiver preenchido, `WEBHOOK_ALREADY_REPLAYED`.
4. Em transação: insere **nova linha** em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, **reaproveitando o payload original** (o snapshot é preservado, não re-renderizado). Preenche `replayedAt` e `replayedById` na linha da DLQ.
5. **Loga quem executou o replay, para auditoria** `[09:36] Sofia`, com `actorId` e `actorEmail` de `req.user`.
6. O worker pega o evento no próximo ciclo, como qualquer outro pendente.

> **Nota de contrato:** o replay gera um `event_id` novo, porque é uma nova linha de outbox. Um cliente que já tenha processado o evento original **não** irá deduplicá-lo pelo `X-Event-Id`. Isso é consistente com a semântica de replay: a operação existe justamente porque a entrega original não foi confirmada.

### 5.5 Rotação de secret

Endpoint: `POST /api/v1/webhooks/:id/rotate-secret` `[09:21] Sofia`.

1. Gera nova secret criptograficamente segura (32 bytes de `crypto.randomBytes`, hex).
2. Em transação: move a secret atual para `previousSecret`, define `previousSecretExpiresAt = now() + 24h`, grava a nova em `secret`.
3. Retorna a nova secret em **texto claro, apenas nesta resposta**.
4. Durante a janela de 24h, o worker assina com a secret **nova**. A `previousSecret` existe para que o cliente aceite requisições assinadas com a antiga enquanto migra os sistemas dele `[09:21] Sofia`. Após o vencimento, uma rotina de limpeza zera `previousSecret`.

---

## 6. Contratos públicos

Todos os endpoints ficam sob o prefixo `/api/v1` já definido em `src/app.ts`, registrados via `buildApiRouter` (`src/routes/index.ts`), e exigem `authenticate` (`src/middlewares/auth.middleware.ts`).

O `customerId` **vem no body ou no path, não do JWT**, porque o JWT atual é do usuário operador da plataforma, não do cliente `[09:32] Bruno`, `[09:32] Larissa`.

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

Cria um endpoint de webhook. A **secret é gerada pela plataforma e devolvida na criação** `[09:31] Marcos`.

**Request**

```json
{
  "customerId": "3f7c1e9a-52b4-4a2d-8f61-9c0d7e4b1a35",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`**

```json
{
  "id": "b1d4f8c2-7e35-4a19-9c8d-2f6a0b3e5147",
  "customerId": "3f7c1e9a-52b4-4a2d-8f61-9c0d7e4b1a35",
  "url": "https://api.atlascomercial.com.br/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "secret": "9f2b7c4e1a8d5306bf9e2c7a4d1b8e5f3072a9c6d4e1b8f5a2c9e6d3b0f7a4c1",
  "active": true,
  "createdAt": "2026-08-01T14:22:03.512Z"
}
```

| Status | Semântica |
| --- | --- |
| `201` | Criado. **Única resposta que expõe a secret em texto claro** |
| `400` | `WEBHOOK_INVALID_URL` (URL não `https`) ou `VALIDATION_ERROR` |
| `401` | Ausência ou invalidez de bearer token |
| `404` | `NOT_FOUND` quando o `customerId` não existe |
| `422` | `WEBHOOK_INVALID_STATUS_FILTER` quando algum status da lista não pertence ao enum `OrderStatus` |

### 6.2 `PATCH /api/v1/webhooks/:id` — editar webhook

Permite alterar URL, filtro de status e estado ativo `[09:33] Bruno`.

**Request**

```json
{
  "url": "https://api.atlascomercial.com.br/v2/webhooks/oms",
  "subscribedStatuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200 OK`**

```json
{
  "id": "b1d4f8c2-7e35-4a19-9c8d-2f6a0b3e5147",
  "customerId": "3f7c1e9a-52b4-4a2d-8f61-9c0d7e4b1a35",
  "url": "https://api.atlascomercial.com.br/v2/webhooks/oms",
  "subscribedStatuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-08-01T15:03:44.118Z"
}
```

| Status | Semântica |
| --- | --- |
| `200` | Atualizado. **Nunca retorna a secret** |
| `400` | `WEBHOOK_INVALID_URL` ou `VALIDATION_ERROR` |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.3 `GET /api/v1/webhooks?customerId=...` — listar webhooks do customer

Lista os webhooks de um customer `[09:33] Bruno`. Usa o helper `paginated()` de `src/shared/http/response.ts`, mesmo contrato `{ data, pagination }` já retornado por `OrderService.list`.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "b1d4f8c2-7e35-4a19-9c8d-2f6a0b3e5147",
      "customerId": "3f7c1e9a-52b4-4a2d-8f61-9c0d7e4b1a35",
      "url": "https://api.atlascomercial.com.br/webhooks/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-01T14:22:03.512Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Semântica |
| --- | --- |
| `200` | Lista paginada. **Secret nunca é retornada** |
| `400` | `VALIDATION_ERROR` quando `customerId` não é UUID |

### 6.4 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

O cliente precisa ver os últimos webhooks enviados, com sucesso ou falha, payload, response e tempo de resposta `[09:34] Marcos`.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "c8e2a6f0-91b7-4d35-8a2e-6f0b4c9d1e73",
      "outboxEventId": "7a3f9c1e-4b82-4d67-9e05-1c8b3a7f2d94",
      "attempt": 1,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 187,
      "sentAt": "2026-08-01T14:31:07.904Z"
    },
    {
      "id": "d9f3b7a1-02c8-4e46-9b3f-7a1c5d0e2f84",
      "outboxEventId": "8b4a0d2f-5c93-4e78-af16-2d9c4b8a3e05",
      "attempt": 2,
      "success": false,
      "responseStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 10000,
      "errorMessage": "upstream returned 503",
      "sentAt": "2026-08-01T14:36:12.441Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

| Status | Semântica |
| --- | --- |
| `200` | Histórico paginado, ordenado por `sentAt` decrescente. Default `pageSize = 100` `[09:34] Marcos` |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.5 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay da DLQ

Exige role `ADMIN` `[09:35] Diego`, `[09:36] Sofia`.

**Request:** sem corpo.

**Response `202 Accepted`**

```json
{
  "deadLetterId": "e0a4c8b2-13d9-4f57-ac48-3e0d5b9a1c26",
  "requeuedOutboxEventId": "f1b5d9c3-24ea-4068-bd59-4f1e6c0b2d37",
  "status": "PENDING",
  "replayedAt": "2026-08-01T16:12:55.203Z",
  "replayedBy": "operador.admin@empresa.com.br"
}
```

| Status | Semântica |
| --- | --- |
| `202` | Aceito e recolocado na outbox. A entrega em si é assíncrona |
| `401` | Sem autenticação |
| `403` | `FORBIDDEN` quando a role não é `ADMIN` `[09:36] Sofia` |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| `409` | `WEBHOOK_ALREADY_REPLAYED` |

### 6.6 `DELETE /api/v1/webhooks/:id` — remover webhook

`[09:33] Bruno`

| Status | Semântica |
| --- | --- |
| `204` | Removido, sem corpo, seguindo o padrão de `OrderController.delete` |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.7 `POST /api/v1/webhooks/:id/rotate-secret` — rotação de secret

`[09:21] Sofia`

**Response `200 OK`**

```json
{
  "id": "b1d4f8c2-7e35-4a19-9c8d-2f6a0b3e5147",
  "secret": "4c1e8b5a2f9d7306ce4b1a8f5d2c9e6b3a0f7d4c1e8b5a2f9d6c3b0a7e4f1d28",
  "previousSecretExpiresAt": "2026-08-02T16:20:11.007Z"
}
```

| Status | Semântica |
| --- | --- |
| `200` | Nova secret gerada. A anterior permanece válida até `previousSecretExpiresAt` (24h) `[09:21] Sofia` |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.8 Contrato de saída: requisição enviada ao cliente

**Headers** `[09:44] Diego`, `[09:44] Sofia`

| Header | Semântica |
| --- | --- |
| `X-Event-Id` | UUID do evento, gerado na inserção na outbox. **Chave de deduplicação no cliente**. Estável entre retentativas `[09:25] Diego` |
| `X-Signature` | HMAC-SHA256 do corpo do request, em hex, com a secret do endpoint `[09:20] Sofia` |
| `X-Timestamp` | Timestamp ISO 8601 do envio. Permite ao cliente detectar replay attack, se quiser `[09:44] Diego` |
| `X-Webhook-Id` | Id do endpoint de webhook, para o cliente que tem vários saber qual cadastro caiu naquele envio `[09:44] Sofia` |
| `Content-Type` | `application/json` `[09:44] Diego` |

**Payload** `[09:43] Diego`

```json
{
  "event_id": "7a3f9c1e-4b82-4d67-9e05-1c8b3a7f2d94",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-01T14:31:07.512Z",
  "order_id": "5c9e2a7f-3b18-4d64-9a05-7e2c1f8b4d63",
  "order_number": "ORD-000142",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f7c1e9a-52b4-4a2d-8f61-9c0d7e4b1a35",
  "total_cents": 148990
}
```

O payload é **deliberadamente enxuto**: não inclui os `items` do pedido, para não inflar. Se o cliente quiser detalhes, ele chama `GET /orders/:id` depois `[09:43] Diego`, `[09:44] Bruno`.

**Resposta esperada do cliente:** qualquer `2xx` é sucesso. Qualquer outra coisa, ou timeout de 10s, é falha e aciona retry `[09:42] Diego`.

### 6.9 Limites de contrato

| Limite | Valor | Origem |
| --- | --- | --- |
| Tamanho máximo do payload | **64KB**, com erro caso ultrapasse | `[09:24] Diego`, `[09:24] Larissa` |
| Timeout da chamada HTTP | **10 segundos** | `[09:42] Diego` |
| Tentativas de entrega | **5** | `[09:15] Diego` |
| Esquema de URL aceito | **`https` apenas** | `[09:23] Sofia` |

---

## 7. Matriz de erros previstos

Todos os erros estendem `AppError` (`src/shared/errors/app-error.ts`) e são serializados pelo `errorMiddleware` existente (`src/middlewares/error.middleware.ts`) no formato `{ error: { code, message, details } }`, **sem nenhuma alteração no middleware** `[09:29] Bruno`. O prefixo `WEBHOOK_` é obrigatório para tudo do módulo `[09:29] Larissa`.

| Código | HTTP | Classe base | Condição | Tratamento |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Endpoint de webhook inexistente `[09:28] Bruno` | Retorna 404, sem efeito colateral |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL não é `https` ou é malformada `[09:23] Sofia`, `[09:28] Bruno` | Recusa no schema Zod, antes de persistir |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `BadRequestError` | Operação exige secret vigente e o endpoint não tem `[09:28] Bruno` | Recusa a operação |
| `WEBHOOK_INVALID_STATUS_FILTER` | 422 | `UnprocessableEntityError` | `subscribedStatuses` contém valor fora do enum `OrderStatus` | Recusa, com `details.invalidStatuses` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload serializado excede 64KB `[09:23] Sofia`, `[09:24] Diego` | **Erra, não trunca.** Se chegou nesse tamanho, tem algo errado `[09:23] Sofia`. Faz a transação de `changeStatus` dar rollback |
| `WEBHOOK_DUPLICATE_URL` | 409 | `ConflictError` | Mesmo customer já tem endpoint ativo com a mesma URL | Recusa, com `details.existingWebhookId` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | Id de DLQ inexistente no replay | Retorna 404 |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `ConflictError` | Linha de DLQ com `replayedAt` preenchido | Recusa o replay duplicado |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | `ConflictError` | Replay ou envio para endpoint com `active = false` | Recusa e registra o motivo |

### 7.1 Erros internos do worker (não expostos via HTTP)

O worker não atende requisições. Estas condições são tratadas no loop de processamento e refletidas em `webhook_outbox.lastError` e em `webhook_deliveries.errorMessage`.

| Condição | Tratamento |
| --- | --- |
| Timeout de 10s na chamada `[09:42] Diego` | Conta como falha, incrementa `attempts`, aplica backoff |
| Resposta não-`2xx` | Conta como falha, grava `responseStatus` e `responseBody` truncado |
| Erro de DNS, TLS ou conexão recusada | Conta como falha, grava `errorMessage` |
| 5ª falha consecutiva | Move para `webhook_dead_letter` `[09:18] Diego` |
| Endpoint desativado ou removido entre a inserção e o envio | Encerra o evento sem tentativa de rede, registrando o motivo |
| Falha do próprio worker (crash) durante `PROCESSING` | Evento fica em `PROCESSING`. Rotina de recuperação no bootstrap devolve para `PENDING` eventos em `PROCESSING` há mais de 60s |

---

## 8. Estratégias de resiliência

| Estratégia | Configuração | Origem |
| --- | --- | --- |
| **Timeout** | 10s por chamada HTTP do worker | `[09:42] Diego` |
| **Retry** | 5 tentativas | `[09:15] Diego` |
| **Backoff exponencial** | 1m, 5m, 30m, 2h, 12h (~15h no total) | `[09:17] Diego` |
| **DLQ** | Tabela `webhook_dead_letter` após a 5ª falha, com payload, motivo e timestamp | `[09:18] Diego` |
| **Replay manual** | Endpoint admin, recoloca na outbox como pendente | `[09:18] Diego` |
| **Atomicidade** | Inserção na outbox dentro da transação de `changeStatus`; rollback conjunto em falha | `[09:40] Bruno`, `[09:41] Diego` |
| **Recuperação de crash** | Eventos em `PROCESSING` há mais de 60s voltam para `PENDING` no bootstrap do worker | Derivado do design de single-worker `[09:11] Diego` |
| **Isolamento de processo** | Worker separado da API; restart da API não afeta a entrega | `[09:11] Diego` |
| **Batch pequeno** | 20 eventos por ciclo, evitando lock longo e uso excessivo de memória | `[09:08] Diego` |

### 8.1 Política de fallback

Não há fallback de canal: **email como alternativa está explicitamente fora desta fase** `[09:37] Larissa`. Quando as 5 tentativas se esgotam, o único caminho é a DLQ e o replay manual por um ADMIN.

Isso significa que **falha permanente de entrega é silenciosa para o cliente**. A mitigação disponível nesta fase é o endpoint de histórico de entregas (6.4), que o cliente pode consultar, e o alerta interno sobre volume de DLQ (seção 9).

### 8.2 Invariantes

1. Se `orders.status` mudou e havia endpoint ativo interessado, existe linha em `webhook_outbox`. Caso contrário, a transação inteira falhou `[09:40] Bruno`.
2. Todo evento converge para `DELIVERED` ou para `webhook_dead_letter`. Não existe estado terminal ambíguo.
3. `attempts` nunca excede 5.
4. Para um mesmo `order_id`, a entrega respeita a ordem de `created_at`, sob single-worker `[09:12] Diego`.
5. `X-Event-Id` é estável entre todas as retentativas do mesmo evento `[09:25] Diego`.
6. A secret em texto claro aparece **apenas** na resposta de criação (6.1) e de rotação (6.7). Nunca em listagem, log ou histórico de entregas.

---

## 9. Observabilidade

### 9.1 Métricas

| Métrica | Tipo | Labels | Finalidade |
| --- | --- | --- | --- |
| `webhook_events_enqueued_total` | Counter | `event_type`, `to_status` | Volume de eventos gerados na outbox |
| `webhook_delivery_attempts_total` | Counter | `result` (`success`/`failure`), `response_status` | Taxa de sucesso de entrega |
| `webhook_delivery_duration_ms` | Histogram | `webhook_endpoint_id` | Latência do endpoint do cliente; detecta cliente lento próximo do timeout de 10s |
| `webhook_outbox_pending_gauge` | Gauge | — | Profundidade da fila. Cresce se o worker está travado |
| `webhook_outbox_oldest_pending_age_seconds` | Gauge | — | **Métrica crítica.** Detecta worker morto, que é o single point of failure do design `[09:11] Diego` |
| `webhook_retry_total` | Counter | `attempt` | Distribuição de retentativas por tentativa |
| `webhook_dead_letter_total` | Counter | `webhook_endpoint_id` | Falhas permanentes por cliente |
| `webhook_replay_total` | Counter | `actor_id` | Volume de replays administrativos, para auditoria `[09:36] Sofia` |
| `webhook_worker_loop_duration_ms` | Histogram | — | Duração do ciclo; se aproximar de 2s, o polling está saturado |

### 9.2 Logs

Usa o logger Pino existente em `src/shared/logger/index.ts`, sem introduzir nada novo `[09:29] Bruno`. Formato JSON estruturado, com `base: { service, env }` e timestamp ISO já configurados.

**Eventos logados:**

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | `info` | `eventId`, `orderId`, `webhookEndpointId`, `toStatus` |
| `webhook_delivery_succeeded` | `info` | `eventId`, `webhookEndpointId`, `attempt`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | `warn` | `eventId`, `webhookEndpointId`, `attempt`, `responseStatus`, `errorMessage`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | `error` | `eventId`, `webhookEndpointId`, `totalAttempts`, `failureReason` |
| `webhook_replay_executed` | `info` | `deadLetterId`, `requeuedOutboxEventId`, `actorId`, `actorEmail` — **obrigatório para auditoria** `[09:36] Sofia` |
| `webhook_secret_rotated` | `info` | `webhookEndpointId`, `previousSecretExpiresAt`. **Nunca o valor da secret** |
| `webhook_worker_started` / `webhook_worker_stopped` | `info` | `pollIntervalMs`, `batchSize` |

**Proteção de dados sensíveis — alteração obrigatória:**

A configuração de `redact` em `src/shared/logger/index.ts` hoje cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`. Ela **precisa ser estendida** para incluir `*.secret`, `*.previousSecret` e `*.signature`.

Essa é a única alteração necessária em código existente fora de `order.service.ts`, e é obrigatória: o incidente de cliente que vazou secret em log de aplicação já aconteceu `[09:22] Diego`.

Regras adicionais:

- Nunca logar o valor da secret, nem a assinatura HMAC calculada.
- `responseBody` do cliente é truncado antes de persistir ou logar.
- Nenhum dado de payload além de identificadores entra em log.

### 9.3 Tracing

O projeto não tem tracing distribuído hoje. A instrumentação mínima proposta para esta feature, mantendo a correlação possível com o `requestId` já produzido por `src/middlewares/request-logger.middleware.ts`:

| Span | Contexto | Atributos |
| --- | --- | --- |
| `webhook.publish` | Dentro da transação de `changeStatus` | `order.id`, `to_status`, `endpoints_matched` |
| `webhook.worker.cycle` | Um por ciclo de polling | `batch_size`, `events_processed` |
| `webhook.delivery.attempt` | Span filho por tentativa de envio | `webhook_endpoint_id`, `attempt`, `response_status`, `duration_ms` |
| `webhook.replay` | Requisição administrativa | `dead_letter_id`, `actor_id` |

**Correlação:** o `eventId` funciona como chave de correlação de ponta a ponta, ligando o log de enfileiramento na API ao log de entrega no worker. Ele é o mesmo valor enviado ao cliente em `X-Event-Id` `[09:25] Diego`, o que permite investigar um caso reportado pelo cliente a partir do id que ele tem em mãos.

**Amostragem:** 100% dos spans de falha e de replay; amostragem reduzida para entregas bem-sucedidas, dado o volume.

### 9.4 Dashboards e alertas mínimos

| Alerta | Condição | Severidade |
| --- | --- | --- |
| Worker parado | `webhook_outbox_oldest_pending_age_seconds > 60` | **Crítica.** Cobre o single point of failure `[09:11] Diego` |
| Fila crescendo | `webhook_outbox_pending_gauge` crescente por 10 minutos | Alta |
| Cliente falhando | Taxa de falha por `webhook_endpoint_id` acima de 50% em 15 minutos | Média. Substitui parcialmente o aviso por email que ficou fora de escopo `[09:37] Larissa` |
| DLQ crescendo | `webhook_dead_letter_total` com incremento em 1 hora | Média |
| Cliente lento | p95 de `webhook_delivery_duration_ms` acima de 8s (perto do timeout de 10s) | Baixa |

---

## 10. Integração com o sistema existente

Esta seção nomeia os arquivos reais do código base e descreve exatamente como o módulo de webhooks se integra a cada um.

### 10.1 `src/modules/orders/order.service.ts` — alteração crítica

É o **único ponto de alteração em fluxo de produção** `[09:40] Bruno`.

O método `changeStatus` (linha 126) já abre `this.prisma.$transaction` e, dentro dela, atualiza `orders`, insere em `order_status_history` e ajusta estoque via `debitStock`/`replenishStock`. A extensão acrescenta **uma chamada** logo após a inserção do histórico, ainda dentro do mesmo `tx`:

```ts
// src/modules/orders/order.service.ts, dentro de changeStatus
await tx.orderStatusHistory.create({
  data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null },
});

// novo: publica o evento na outbox dentro da MESMA transação
await publishWebhookEvent(tx, order, from, to);
```

A forma escolhida é uma **função que recebe o `tx` client**, não a injeção de um repository de webhook no construtor do `OrderService` `[09:41] Bruno`, `[09:41] Diego`. O `OrderService` hoje recebe apenas `OrderRepository` e `PrismaClient`; manter essa assinatura evita propagar dependência de webhook por todo o wiring de `buildControllers` em `src/app.ts`.

**Consequência transacional:** se `publishWebhookEvent` lançar (por exemplo `WEBHOOK_PAYLOAD_TOO_LARGE`), o `$transaction` faz rollback de tudo, incluindo a mudança de status e o ajuste de estoque. É o comportamento exigido: não pode ter caso de status mudar e evento não sair `[09:40] Bruno`, `[09:41] Diego`.

### 10.2 `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — reuso sem alteração

`AppError` (`src/shared/errors/app-error.ts`) expõe `statusCode`, `errorCode` e `details`. As classes de `src/shared/errors/http-errors.ts` (`NotFoundError`, `BadRequestError`, `ConflictError`, `UnprocessableEntityError`) seguem o padrão em que erros de domínio especializam um erro HTTP, como `InsufficientStockError extends UnprocessableEntityError` e `InvalidStatusTransitionError extends ConflictError`.

Os erros de webhook seguem exatamente esse padrão, em arquivo próprio do módulo:

```ts
// src/modules/webhooks/webhook.errors.ts
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook'); }   // produz WEBHOOK_NOT_FOUND via override de código
}

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(sizeBytes: number) {
    super('Webhook payload exceeds the 64KB limit', 'WEBHOOK_PAYLOAD_TOO_LARGE', { sizeBytes, limitBytes: 65536 });
  }
}
```

Nenhum arquivo de `src/shared/errors/` é modificado. O barrel `src/shared/errors/index.ts` continua exportando o que já exporta; os erros do módulo são exportados pelo próprio módulo `[09:28] Bruno`.

### 10.3 `src/middlewares/error.middleware.ts` — zero alteração

O `errorMiddleware` já trata `AppError` no primeiro branch, serializando `{ error: { code, message, details } }`, e também `ZodError` e `Prisma.PrismaClientKnownRequestError`.

Como todos os erros de webhook estendem `AppError`, **o middleware os captura sem nenhuma mudança** `[09:29] Bruno`. Isso vale também para as validações de schema: o `validate` middleware converte `ZodError` em `ValidationError` antes de chegar ao handler.

### 10.4 `src/middlewares/auth.middleware.ts` — reuso de `authenticate` e `requireRole`

Todas as rotas do módulo usam `router.use(authenticate)`, exatamente como `src/modules/orders/order.routes.ts` faz.

O endpoint de replay adiciona `requireRole('ADMIN')` `[09:36] Larissa`. A função já existe e valida contra `AuthUser['role']`, que é `'ADMIN' | 'OPERATOR'`, lançando `ForbiddenError` quando a role não bate:

```ts
// src/modules/webhooks/webhook.routes.ts
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  requireRole('ADMIN'),
  validate({ params: deadLetterIdParamSchema }),
  controller.replayDeadLetter,
);
```

O `req.user` populado por `authenticate` fornece `id` e `email` para o log de auditoria do replay `[09:36] Sofia`.

O restante do CRUD fica apenas com `authenticate`, aceitando qualquer role autenticada por enquanto `[09:37] Sofia`.

### 10.5 `src/shared/logger/index.ts` — reuso com extensão obrigatória de `redact`

O logger Pino é reaproveitado sem substituição `[09:29] Bruno`. A única alteração é acrescentar os caminhos de secret ao array `redactPaths` já existente, conforme detalhado em 9.2. Sem isso, há risco concreto de vazamento de secret em log `[09:22] Diego`.

### 10.6 `prisma/schema.prisma` — aditivo

As quatro tabelas da seção 4 são acrescentadas seguindo as convenções do arquivo. `WebhookEndpoint` cria relação com o `Customer` existente, o que exige acrescentar o campo inverso `webhookEndpoints WebhookEndpoint[]` no model `Customer`. Nenhuma tabela existente tem coluna alterada ou removida, portanto a migração é puramente aditiva.

### 10.7 `src/config/database.ts` — reuso do factory pelo worker

O worker chama `createPrismaClient()` para obter **instância própria** de `PrismaClient`, porque o client é por processo `[09:30] Bruno`. Mesma `DATABASE_URL`, sem alteração em `src/config/env.ts` além das variáveis novas de configuração do worker (seção 11).

### 10.8 `src/routes/index.ts` e `src/app.ts` — registro do módulo

`buildApiRouter` em `src/routes/index.ts` passa a montar o router de webhooks, seguindo o mesmo padrão dos módulos existentes:

```ts
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
```

O type `Controllers` recebe a entrada `webhooks: WebhookController`, e `buildControllers` em `src/app.ts` instancia repository, service e controller na mesma sequência já usada para os outros módulos. O prefixo `/api/v1` continua sendo aplicado em `src/app.ts`.

### 10.9 `src/modules/orders/order.status.ts` — reuso do enum de status

O filtro `subscribedStatuses` é validado contra o enum `OrderStatus` do Prisma, o mesmo usado por `canTransition` e por `updateOrderStatusSchema`. O módulo de webhooks **não redefine** a lista de status válidos, o que garante que um status novo na máquina de estados não crie divergência silenciosa no filtro.

### 10.10 `src/modules/orders/order.routes.ts` e `order.schemas.ts` — referência de padrão

Servem como modelo estrutural para `webhook.routes.ts` e `webhook.schemas.ts`: `router.use(authenticate)` no topo, `validate({ params, body, query })` por rota, schemas Zod exportados com os tipos inferidos via `z.infer`.

### 10.11 `tests/orders.test.ts` e `tests/helpers/factories.ts` — base de teste

Os testes do módulo seguem o padrão existente, reaproveitando as factories de `tests/helpers/factories.ts` e o setup de `tests/setup.ts`. O teste crítico a acrescentar é de integração no fluxo de `changeStatus`, verificando o invariante de atomicidade da seção 8.2.

---

## 11. Dependências e compatibilidade

### 11.1 Dependências de runtime

| Componente | Versão | Situação | Observação |
| --- | --- | --- | --- |
| Node.js | `>=20` | Já no projeto (`package.json`) | `fetch` e `AbortSignal.timeout` nativos atendem o timeout de 10s, sem dependência HTTP nova |
| `@prisma/client` / `prisma` | 5.22.0 | Já no projeto | `Prisma.TransactionClient` é o tipo do `tx` recebido por `publishWebhookEvent` |
| MySQL | Conforme `docker-compose.yml` | Já no projeto | Sem recurso novo do engine. Sem `NOTIFY`/`LISTEN` `[09:09] Diego` |
| `zod` | 3.23.8 | Já no projeto | Validação de URL `https` e do filtro de status |
| `pino` | 9.5.0 | Já no projeto | Logger reaproveitado `[09:29] Bruno` |
| `uuid` | 11.0.3 | Já no projeto | Geração do `event_id`; alternativamente `crypto.randomUUID` |
| `node:crypto` | Nativo | — | `createHmac('sha256', ...)` para a assinatura e `randomBytes` para a secret |

**Nenhuma dependência nova de terceiro é necessária**, e nenhuma infraestrutura nova é provisionada `[09:07] Diego`, `[09:29] Bruno`.

### 11.2 Configuração nova

Acrescentada a `src/config/env.ts` seguindo o padrão de schema Zod já usado no arquivo, e a `.env.example`:

| Variável | Default | Finalidade |
| --- | --- | --- |
| `WEBHOOK_WORKER_POLL_INTERVAL_MS` | `2000` | Intervalo de polling `[09:09] Diego` |
| `WEBHOOK_WORKER_BATCH_SIZE` | `20` | Tamanho do batch `[09:08] Diego` |
| `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | Timeout da chamada `[09:42] Diego` |
| `WEBHOOK_MAX_ATTEMPTS` | `5` | Teto de tentativas `[09:15] Diego` |
| `WEBHOOK_MAX_PAYLOAD_BYTES` | `65536` | Limite de 64KB `[09:24] Diego` |
| `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` | `24` | Grace period da rotação `[09:21] Sofia` |

### 11.3 Novo script de execução

Em `package.json`, acompanhando o padrão de `dev` e `start` `[09:11] Larissa`:

```json
"worker": "tsx watch --env-file=.env src/worker.ts",
"worker:start": "node --env-file=.env dist/worker.js"
```

### 11.4 Garantias de compatibilidade

- **Nenhum contrato existente muda.** Os endpoints de `/orders`, `/customers`, `/products`, `/users` e `/auth` mantêm request e response idênticos.
- `PATCH /orders/:id/status` mantém o mesmo contrato de entrada e saída. O que muda é um efeito colateral interno.
- **Novo modo de falha em `changeStatus`:** um pedido pode passar a falhar com `WEBHOOK_PAYLOAD_TOO_LARGE` (422), o que não acontecia antes. É uma consequência aceita da exigência de atomicidade `[09:41] Diego`, e a probabilidade é baixa porque o payload é enxuto e nenhum evento chega perto de 64KB `[09:24] Diego`.
- Migração Prisma puramente aditiva, sem alteração destrutiva.
- API versionada em `/api/v1`, como já é.

---

## 12. Critérios de aceite técnicos

### Atomicidade e integridade

- [ ] Mudança de status com endpoint ativo interessado gera exatamente uma linha em `webhook_outbox` por endpoint, na mesma transação `[09:40] Bruno`.
- [ ] Falha forçada na inserção da outbox faz rollback da mudança de status e do ajuste de estoque, verificado por teste de integração.
- [ ] Mudança para status que **nenhum** endpoint assinou não gera linha na outbox `[09:34] Bruno`.
- [ ] Payload persistido é snapshot: alterar o pedido depois não altera o payload do evento já enfileirado `[09:52] Larissa`.

### Entrega e retry

- [ ] Evento pendente é entregue em até 2s + latência do cliente, medido do commit até a primeira tentativa `[09:09] Diego`.
- [ ] Resposta `2xx` marca o evento como `DELIVERED` e registra a entrega em `webhook_deliveries`.
- [ ] Falha aplica a progressão exata 1m/5m/30m/2h/12h `[09:17] Diego`.
- [ ] Cliente que não responde em 10s é tratado como falha `[09:42] Diego`.
- [ ] 5ª falha move o evento para `webhook_dead_letter` com payload, motivo e `totalAttempts` `[09:18] Diego`.
- [ ] `X-Event-Id` é idêntico em todas as retentativas do mesmo evento `[09:25] Diego`.
- [ ] Três mudanças de status sequenciais no mesmo pedido chegam ao cliente na ordem correta `[09:12] Diego`.

### Segurança

- [ ] `X-Signature` valida como HMAC-SHA256 do corpo com a secret do endpoint, verificado por teste que recalcula a assinatura `[09:20] Sofia`.
- [ ] Cada endpoint tem secret distinta; nenhuma secret é compartilhada entre endpoints `[09:21] Sofia`.
- [ ] URL `http` é recusada com `WEBHOOK_INVALID_URL` `[09:23] Sofia`.
- [ ] Após rotação, requisições assinadas com a secret anterior continuam verificáveis por 24h `[09:21] Sofia`.
- [ ] Secret aparece em texto claro **apenas** nas respostas de criação e rotação; nunca em listagem, histórico ou log.
- [ ] Replay sem role `ADMIN` retorna `403` `[09:36] Sofia`.
- [ ] Replay bem-sucedido emite log com `actorId` e `actorEmail` `[09:36] Sofia`.
- [ ] Nenhum log contém valor de secret ou assinatura, verificado com `redact` ativo `[09:22] Diego`.

### Contratos e erros

- [ ] Os 7 endpoints da seção 6 respondem com os status codes especificados.
- [ ] Todos os códigos de erro do módulo usam prefixo `WEBHOOK_` `[09:29] Larissa`.
- [ ] Payload excedendo 64KB retorna erro, sem truncar `[09:23] Sofia`.
- [ ] Erros de webhook são serializados pelo `errorMiddleware` existente, sem alteração no middleware `[09:29] Bruno`.
- [ ] Payload de saída contém exatamente os campos da seção 6.8, sem `items` `[09:43] Diego`.

### Observabilidade

- [ ] `webhook_outbox_oldest_pending_age_seconds` reflete o evento pendente mais antigo, e o alerta dispara com o worker parado.
- [ ] Todo evento do ciclo de vida produz log estruturado com o `eventId` como chave de correlação.

### Testes obrigatórios

- [ ] **Unitários:** cálculo de HMAC, progressão do backoff, filtro de status, validação de tamanho de payload.
- [ ] **Integração:** transação de `changeStatus` com outbox (incluindo rollback), ciclo completo do worker contra servidor HTTP de teste, movimentação para DLQ e replay.
- [ ] **Segurança:** revisão dedicada da Sofia sobre HMAC e geração de secret, com no mínimo **dois dias úteis** reservados antes do deploy `[09:46] Sofia`.

---

## 13. Riscos e mitigação

### 13.1 Worker único cai e a entrega para silenciosamente

- **Probabilidade:** média
- **Impacto:** alto. A feature inteira para de funcionar, e nada no fluxo de pedidos sinaliza o problema. É o single point of failure aceito no design `[09:11] Diego`, `[09:12] Diego`.
- **Mitigação:**
  - Alerta crítico em `webhook_outbox_oldest_pending_age_seconds > 60`.
  - Rotina de recuperação no bootstrap que devolve eventos travados em `PROCESSING` para `PENDING`.
  - Supervisor de processo com restart automático.
- **Plano de contingência:** subir o worker manualmente. Como a outbox é durável, nenhum evento é perdido durante a indisponibilidade: eles são entregues em backlog quando o worker volta.

### 13.2 Falha na inserção da outbox derruba mudança de status de pedido

- **Probabilidade:** baixa
- **Impacto:** alto. Bloqueia a operação central da plataforma, não apenas a notificação.
- **Mitigação:**
  - A operação é `INSERT` local, sem I/O de rede dentro da transação.
  - Payload enxuto mantém o risco de `WEBHOOK_PAYLOAD_TOO_LARGE` remoto `[09:43] Diego`.
  - Teste de integração cobrindo o caminho de rollback.
- **Plano de contingência:** desativar os endpoints de webhook do customer afetado (`active = false`), o que faz `publishWebhookEvent` retornar sem inserir e restaura o fluxo de pedidos imediatamente.

### 13.3 Vazamento de secret de webhook

- **Probabilidade:** média. Já ocorreu com um cliente da plataforma `[09:22] Diego`.
- **Impacto:** alto. Permite a um terceiro forjar eventos que o cliente aceitará como legítimos.
- **Mitigação:**
  - Secret por endpoint contém o blast radius `[09:21] Sofia`.
  - Rotação com grace period de 24h, sem downtime da integração `[09:21] Sofia`.
  - `redact` estendido no logger (9.2).
  - Secret exposta em texto claro apenas em criação e rotação.
  - Revisão de segurança dedicada antes do deploy `[09:46] Sofia`.
- **Plano de contingência:** rotação imediata da secret do endpoint afetado. **Limitação conhecida:** o grace period mantém a secret comprometida válida por até 24h, e não existe operação de revogação imediata nesta entrega. Registrado como questão em aberto no [RFC](./RFC.md#53-revogação-imediata-de-secret-comprometida).

### 13.4 Crescimento indefinido da outbox

- **Probabilidade:** alta. É consequência certa do design, já que o arquivamento está fora de escopo `[09:08] Diego`.
- **Impacto:** médio. Degradação progressiva da query de polling.
- **Mitigação:**
  - Índice composto `[status, nextAttemptAt]`, que mantém a leitura dos pendentes barata independentemente do volume de entregues.
  - Batch pequeno `[09:08] Diego`.
  - Monitoramento do tamanho da tabela.
- **Plano de contingência:** arquivamento manual de linhas `DELIVERED` antigas até que a rotina automática seja implementada em fase futura.

### 13.5 Cliente não implementa deduplicação por `X-Event-Id`

- **Probabilidade:** média
- **Impacto:** médio. Efeito colateral duplicado no sistema do cliente, com potencial de disputa comercial.
- **Mitigação:**
  - Documentação destacada no portal do desenvolvedor `[09:26] Marcos`.
  - `X-Event-Id` estável entre retentativas, o que torna a deduplicação trivial de implementar `[09:25] Diego`.
- **Plano de contingência:** endpoint de histórico de entregas (6.4) permite reconstruir com o cliente exatamente o que foi enviado e quantas vezes.

### 13.6 Cliente lento degrada o throughput do worker

- **Probabilidade:** média
- **Impacto:** médio. Com single-worker processando sequencialmente, um cliente que consome os 10s de timeout atrasa a entrega para todos os outros.
- **Mitigação:**
  - Timeout de 10s como teto rígido `[09:42] Diego`.
  - Métrica `webhook_delivery_duration_ms` por endpoint, com alerta em p95 acima de 8s.
- **Plano de contingência:** desativar temporariamente o endpoint problemático. A escala do worker para múltiplos processos é o caminho estrutural, adiado como problema do futuro `[09:13] Diego`.
