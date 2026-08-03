# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma)
- **Origem:** Reunião técnica de webhooks, `[09:27]` a `[09:30]`

## Contexto

O módulo de webhooks é a primeira feature da plataforma que introduz processamento assíncrono, worker e comunicação outbound. Havia risco real de o módulo divergir estruturalmente do resto da aplicação, criando um segundo conjunto de convenções dentro da mesma codebase.

A codebase tem um padrão claro e consistente, levantado na reunião: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas `[09:27] Bruno`.

A posição do time foi fechar isso como decisão explícita: **reuso máximo do que já existe**, com o webhook sendo um módulo igual aos outros `[09:30] Larissa`.

## Decisão

O módulo de webhooks segue as convenções já estabelecidas no projeto, sem introduzir infraestrutura ou padrão novo. Cada item abaixo referencia o artefato real do código base que será reaproveitado.

### 1. Estrutura modular

Nova pasta `src/modules/webhooks` replicando a estrutura dos módulos existentes `[09:27] Bruno`, tal como `src/modules/orders/` a exibe hoje:

| Arquivo de referência existente | Equivalente no módulo de webhooks |
| --- | --- |
| `src/modules/orders/order.controller.ts` | `webhook.controller.ts` |
| `src/modules/orders/order.service.ts` | `webhook.service.ts` |
| `src/modules/orders/order.repository.ts` | `webhook.repository.ts` |
| `src/modules/orders/order.routes.ts` | `webhook.routes.ts` |
| `src/modules/orders/order.schemas.ts` | `webhook.schemas.ts` |

O registro do router acompanha o padrão de `src/routes/index.ts`, que hoje monta os módulos via `buildApiRouter` sob o prefixo `/api/v1` definido em `src/app.ts`.

A lógica de processamento do worker fica em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`), com a entry-point separada em `src/worker.ts` `[09:28] Bruno`, conforme [ADR-005](./ADR-005-worker-processo-separado-polling.md).

### 2. Classes de erro e padrão de códigos

O projeto já tem hierarquia de erro estabelecida em `src/shared/errors/app-error.ts` (classe base `AppError`, com `statusCode`, `errorCode` e `details`) e `src/shared/errors/http-errors.ts` (`NotFoundError`, `ValidationError`, `ConflictError`, `UnprocessableEntityError`, além de erros de domínio como `InsufficientStockError` e `InvalidStatusTransitionError`) `[09:28] Bruno`.

O módulo de webhooks **estende `AppError`** seguindo o mesmo formato, e adota o padrão de códigos em SCREAMING_SNAKE_CASE já usado por `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`, com **prefixo `WEBHOOK_` para tudo do módulo** `[09:29] Larissa`: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, entre outros `[09:28] Bruno`. A matriz completa está no FDD.

### 3. Middleware de erro centralizado, sem alteração

O `errorMiddleware` em `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError`, serializando a resposta no formato `{ error: { code, message, details } }`.

Como os erros de webhook estendem `AppError`, **o middleware pega os erros novos sem precisar de nenhuma mudança** `[09:29] Bruno`.

### 4. Logger Pino existente

O logger em `src/shared/logger/index.ts` já está em uso no projeto inteiro. **Nada novo é introduzido** `[09:29] Bruno`.

Ponto relevante para esta feature: o logger já aplica `redact` em caminhos sensíveis, incluindo `*.token`, `*.password` e `req.headers.authorization`, com censor `[REDACTED]`. A configuração de redact precisa ser estendida para cobrir a secret do webhook, conforme especificado na seção de observabilidade do FDD.

### 5. Validação com Zod

Os schemas seguem o padrão de `src/modules/orders/order.schemas.ts`, consumidos pelo middleware `validate` de `src/middlewares/validate.middleware.ts`, que já converte `ZodError` em `ValidationError` com `path` e `message` por issue.

A validação de URL `https` obrigatória ([ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md)) entra como regra de schema Zod, conforme classificado na reunião `[09:23] Sofia`.

### 6. Autenticação e autorização existentes

Reaproveita `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts`. O `requireRole('ADMIN')` é usado no endpoint de replay da DLQ `[09:36] Larissa`, exatamente como o padrão já existente prevê (`AuthUser['role']` sendo `'ADMIN' | 'OPERATOR'`).

### 7. Prisma e modelagem

As tabelas novas são declaradas em `prisma/schema.prisma`, seguindo as convenções já vigentes no arquivo: `id` em `String @id @default(uuid()) @db.Char(36)`, mapeamento explícito de nome de tabela com `@@map`, e `@@index` nos campos de consulta. O uso de UUID acompanha o padrão do resto do projeto `[09:51] Larissa`.

O worker instancia `PrismaClient` próprio, seguindo o factory `createPrismaClient()` de `src/config/database.ts`, porque `PrismaClient` é por processo `[09:30] Bruno`.

### 8. Formato de resposta paginada

Os endpoints de listagem (webhooks de um customer, histórico de entregas) usam o helper `paginated()` de `src/shared/http/response.ts`, mesmo contrato `{ data, pagination }` já retornado por `OrderService.list`.

## Alternativas Consideradas

### Módulo com estrutura própria, desacoplado das convenções existentes

Tratar webhooks como subsistema independente, com organização interna própria por ser a primeira feature assíncrona.

- **Descartada.** Criaria dois conjuntos de convenções na mesma codebase, elevando o custo de manutenção e a curva de entrada para qualquer dev que transite entre módulos. A decisão da reunião foi explicitamente a oposta: webhook fica como módulo igual aos outros `[09:30] Larissa`.
- **Trade-off do descarte:** uma estrutura própria poderia acomodar melhor as particularidades do processamento assíncrono, que os módulos CRUD existentes não têm. Perdeu para consistência da codebase. O ponto de fuga necessário (o worker) foi resolvido de forma mínima, com uma entry-point nova em `src/worker.ts` e a lógica ainda dentro do módulo `[09:28] Bruno`.

### Biblioteca de erros nova para o domínio de webhooks

Criar hierarquia de erro dedicada em vez de estender `AppError`.

- **Descartada.** Exigiria alteração no `errorMiddleware` de `src/middlewares/error.middleware.ts` para reconhecer o tipo novo. Estendendo `AppError`, o middleware funciona sem mudança alguma `[09:29] Bruno`.
- **Trade-off do descarte:** uma hierarquia dedicada poderia modelar melhor estados específicos de entrega. Perdeu para custo zero de integração com o tratamento de erro existente.

### Logger dedicado para o worker

Introduzir instrumentação própria para o processo assíncrono.

- **Descartada.** O Pino já está no projeto inteiro e a diretriz foi não botar nada novo `[09:29] Bruno`.
- **Trade-off do descarte:** um logger dedicado permitiria configuração de nível e destino independentes para o worker. Perdeu para uniformidade; o `createLogger()` existente já aceita configuração por env (`LOG_LEVEL`), o que cobre a necessidade.

## Consequências

### Positivas

- Curva de entrada baixa: qualquer dev que conhece `src/modules/orders/` sabe navegar `src/modules/webhooks/`.
- **Nenhuma alteração necessária** em `src/middlewares/error.middleware.ts`, `src/shared/errors/`, `src/middlewares/auth.middleware.ts` ou `src/shared/logger/index.ts`. Reduz a superfície de regressão da feature sobre o sistema existente.
- Prefixo `WEBHOOK_` torna imediato, ao ler um log ou uma resposta de erro, se o problema é do módulo de webhooks `[09:29] Larissa`.
- Testes seguem o padrão já existente em `tests/orders.test.ts`, com os helpers de `tests/helpers/factories.ts`.

### Negativas

- O padrão de módulo CRUD existente não tem nada equivalente a um worker de processamento contínuo. Forçar o worker dentro dessa estrutura é uma adaptação, não um encaixe natural: a entry-point `src/worker.ts` fica fora de `src/modules/`, enquanto a lógica fica dentro, o que divide o componente em dois lugares `[09:11] Larissa`, `[09:28] Bruno`.
- A configuração de `redact` do logger em `src/shared/logger/index.ts` **precisa ser estendida** para cobrir a secret do webhook. É a única exceção ao "não mudar nada do existente", e é obrigatória: sem isso há risco de vazamento de secret em log, exatamente o incidente que já ocorreu com um cliente `[09:22] Diego`.
- Herdar as convenções significa herdar suas limitações. O `errorMiddleware` retorna `500` genérico para erro não mapeado, o que exige disciplina em mapear todos os estados de falha de entrega como `AppError`.

### Trade-off explícito

Trocamos **liberdade de modelar o subsistema assíncrono da forma ideal** por **consistência e custo zero de integração com o que já existe**. Dado que o time é pequeno `[09:07] Diego` e que a feature tem prazo de três sprints `[09:47] Larissa`, a consistência vale mais do que a modelagem ideal.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./ADR-001-outbox-no-mysql.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](./ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Worker em processo separado com polling](./ADR-005-worker-processo-separado-polling.md)
