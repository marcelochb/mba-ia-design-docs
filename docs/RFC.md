# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-08-01 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| **Origem** | Reunião técnica de webhooks (`TRANSCRICAO.md`), quinta-feira 09:00, ~55 min |
| **Documentos relacionados** | [PRD](./PRD.md), [FDD](./FDD.md), [ADRs](./adrs/), [Tracker](./TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos construir um sistema de **webhooks outbound** que notifica clientes B2B quando o status de um pedido muda, eliminando a necessidade de polling em `GET /orders`.

A abordagem é o **padrão Outbox no MySQL existente**: a mudança de status grava o evento na mesma transação que atualiza o pedido, e um **worker em processo separado** consome essa tabela por polling de 2 segundos e dispara as chamadas HTTP. A entrega é **at-least-once**, autenticada por **HMAC-SHA256** com secret por endpoint, com **retry em backoff exponencial** (5 tentativas ao longo de ~15h) e **DLQ persistida** para falhas permanentes.

A escolha central é deliberadamente conservadora: nenhuma infraestrutura nova. O requisito de latência acordado com os clientes é frouxo (abaixo de 10 segundos é considerado tempo real `[09:02] Marcos`), o que torna um broker de mensageria desnecessário para um time pequeno `[09:07] Diego`.

**Estimativa:** três sprints, incluindo a revisão de segurança `[09:47] Larissa`.

---

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fizeram pedido formal para serem notificados em tempo real quando o status dos pedidos deles muda `[09:00] Marcos`.

Hoje esses clientes fazem polling em `GET /orders` de tempos em tempos para descobrir se algo mudou. Isso deixa a integração deles lenta e cara `[09:00] Marcos`.

O problema tem urgência comercial: a Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre `[09:00] Marcos`.

A aplicação atual é um Order Management System em Node.js e TypeScript, com MySQL via Prisma. Ela **não tem nenhum mecanismo de notificação externa, evento, fila ou webhook**. O ciclo de vida do pedido tem máquina de estados controlada (`src/modules/orders/order.status.ts`) e mudança de status transacional (`src/modules/orders/order.service.ts`), mas nada disso é observável de fora.

**Escopo delimitado na reunião:** os webhooks são apenas **outbound**, saindo da plataforma para os clientes. Os clientes querem receber, não enviar `[09:02] Marcos`, `[09:02] Sofia`, o que simplifica o problema `[09:03] Sofia`.

**Requisito de latência:** os clientes consideram tempo real qualquer coisa abaixo de 10 segundos. O que importa para eles é não ficar pendurado atualizando manualmente `[09:02] Marcos`.

---

## 3. Proposta técnica

### 3.1 Visão geral

```
┌─────────────────────────┐
│  API (src/server.ts)    │
│                         │
│  PATCH /orders/:id/     │
│        status           │
│         │               │
│         ▼               │
│  OrderService           │
│  .changeStatus()        │
│  ┌───────────────────┐  │
│  │ TRANSAÇÃO ÚNICA   │  │
│  │ • update order    │  │
│  │ • insert history  │  │
│  │ • ajusta estoque  │  │
│  │ • INSERT outbox ◄─┼──┼── novo
│  └───────────────────┘  │
└─────────────────────────┘
              │
              ▼
       ┌──────────────┐
       │ MySQL        │
       │ webhook_     │
       │  outbox      │
       │ webhook_     │
       │  dead_letter │
       └──────────────┘
              │ polling 2s
              ▼
┌─────────────────────────┐          ┌──────────────────┐
│ Worker (src/worker.ts)  │  HTTPS   │ Endpoint do      │
│ processo separado       │ ───────► │ cliente          │
│ • lê pendentes em batch │  HMAC    │ (Atlas, Max,     │
│ • assina HMAC-SHA256    │  10s TO  │  Nova Cargo)     │
│ • POST, timeout 10s     │          └──────────────────┘
│ • retry / DLQ           │
└─────────────────────────┘
```

### 3.2 Padrão Outbox transacional

Quando o status do pedido muda, **dentro da mesma transação SQL** que atualiza `orders` e insere em `order_status_history`, uma linha de evento é inserida em `webhook_outbox` `[09:06] Diego`.

A garantia que isso dá: se a transação principal commitou, o evento foi registrado; se deu rollback, o evento desaparece junto `[09:06] Diego`. O requisito é explícito: não pode existir caso de status mudar e evento não sair `[09:40] Bruno`.

Descartamos o disparo síncrono porque a transação de mudança de status já é pesada (atualiza pedido, insere histórico, decrementa estoque) e uma chamada HTTP no meio dela faria um cliente lento travar a operação de outros pedidos `[09:04] Bruno`. Além disso, não haveria tratamento razoável para cliente fora do ar: rollback do status não é opção `[09:04] Bruno`.

O evento guarda o **payload já renderizado** (snapshot na inserção), para que uma mudança posterior no pedido não altere o conteúdo do evento já disparado `[09:52] Larissa`.

Detalhe: o **filtro de eventos é aplicado na inserção**. Se nenhum webhook do customer quer aquele status, a linha não é inserida, economizando espaço na tabela `[09:34] Bruno`, `[09:34] Diego`.

Ver [ADR-001](./adrs/ADR-001-outbox-no-mysql.md).

### 3.3 Worker em processo separado

O consumo é feito por **polling de 2 segundos** em batch pequeno dos eventos pendentes mais antigos `[09:09] Diego`. Isso atende o requisito de 10 segundos com folga, ao custo de uma latência mínima de 2 segundos, aceita explicitamente `[09:10] Larissa`.

O worker roda como **processo separado** da API, com entry-point própria (`src/worker.ts`, à semelhança de `src/server.ts`) e script `npm run worker` `[09:11] Larissa`. Se rodasse dentro da API, um restart derrubaria o worker `[09:11] Diego`. Mesmo banco e mesma stack, com `PrismaClient` próprio por ser outro processo `[09:30] Bruno`.

**Ordering:** com single-worker, os eventos são processados por `created_at` e o cliente recebe em ordem `[09:12] Diego`. Fica documentado como limitação conhecida: **não há garantia de ordering global**, apenas por `order_id` e enquanto for single-worker `[09:13] Larissa`. Os clientes nunca pediram ordering global `[09:14] Marcos`.

Ver [ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md).

### 3.4 Entrega, retry e DLQ

- **Timeout de 10 segundos** por chamada. Cliente que não responde nesse prazo é tratado como falha `[09:42] Diego`.
- **5 tentativas** com backoff exponencial de **1m, 5m, 30m, 2h, 12h**, totalizando ~15 horas `[09:17] Diego`.
- Esgotadas as tentativas, o evento vai para **DLQ em tabela separada** (`webhook_dead_letter`), com payload, motivo da falha e timestamp `[09:18] Diego`.
- Reprocessamento é **manual, via endpoint administrativo** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente `[09:18] Diego`.

A entrega é **at-least-once**: o cliente pode receber o mesmo evento duas vezes e deve deduplicar pelo `X-Event-Id`, um UUID gerado na inserção na outbox `[09:24] Diego`, `[09:25] Diego`.

Ver [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md) e [ADR-003](./adrs/ADR-003-at-least-once-x-event-id.md).

### 3.5 Segurança

- **HMAC-SHA256** sobre o corpo do request, enviado no header `X-Signature` `[09:20] Sofia`.
- **Secret única por endpoint**, não global. Se vazar uma, não vaza tudo `[09:21] Sofia`. A secret é gerada pela plataforma e devolvida na criação do webhook `[09:31] Marcos`.
- **Rotação com grace period de 24h**: a secret antiga continua válida em paralelo por 24 horas para o cliente migrar `[09:21] Sofia`.
- **TLS obrigatório**: URL precisa ser `https`, `http` é recusado com erro de validação `[09:23] Sofia`.
- **Role `ADMIN`** obrigatório no endpoint de replay, reaproveitando o `requireRole` existente, com log de quem executou o replay para auditoria `[09:36] Sofia`, `[09:36] Larissa`.

Ver [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).

### 3.6 Reuso do que já existe

O módulo entra como `src/modules/webhooks`, seguindo a estrutura dos módulos existentes (controller, service, repository, routes, schemas) `[09:27] Bruno`. Reaproveita `AppError`, o Pino, o error middleware centralizado, o padrão de schemas Zod e o padrão de códigos de erro, agora com **prefixo `WEBHOOK_`** `[09:29] Larissa`, `[09:30] Larissa`.

Nenhuma infraestrutura nova é introduzida `[09:29] Bruno`.

Ver [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md).

---

## 4. Alternativas consideradas

### 4.1 Disparo HTTP síncrono no `changeStatus`

Chamar o endpoint do cliente diretamente dentro da transação de mudança de status.

- **Trade-off que motivou o descarte:** acopla a latência e a disponibilidade da operação de pedidos ao endpoint do cliente. A transação já é pesada, e um cliente lento travaria a mudança de status de outros pedidos `[09:04] Bruno`. Pior, não há tratamento consistente para falha do cliente: dar rollback na mudança de status porque a notificação falhou é inaceitável `[09:04] Bruno`. Foi classificado como fora de questão `[09:06] Diego`.

### 4.2 Redis Streams ou broker de mensageria dedicado

Publicar o evento em Redis Streams e consumir de lá.

- **Trade-off que motivou o descarte:** exigiria subir infraestrutura nova. Para um time pequeno, subir Redis Cluster para isso é overengineering, e o outbox no MySQL existente resolve `[09:07] Larissa`, `[09:07] Diego`. Abrimos mão do throughput e do fan-out nativo de um broker em troca de zero custo operacional adicional e de atomicidade transacional gratuita.

### 4.3 Trigger de banco para consumo reativo

Usar trigger no MySQL para notificar o worker no instante da inserção, em vez de polling.

- **Trade-off que motivou o descarte:** o MySQL não tem listener nativo tipo `NOTIFY`/`LISTEN` do Postgres. A trigger só executa SQL, não notifica processo externo. Avisar o worker exigiria improvisar algo como escrever em arquivo ou bater em um endpoint, o que foi considerado esquisito `[09:09] Diego`. O polling de 2 segundos atende o requisito de 10 segundos tranquilamente `[09:09] Diego`.

### 4.4 3 tentativas de retry em vez de 5

Política mais agressiva no descarte de eventos.

- **Trade-off que motivou o descarte:** três tentativas cobririam ~30 minutos. Um cliente com indisponibilidade matinal teria os eventos mortos antes de voltar, e já houve cliente da plataforma com indisponibilidade de duas horas em manutenção planejada `[09:16] Diego`. Aceitamos manter a linha ocupada por mais tempo em troca de cobrir janelas reais de indisponibilidade.

### 4.5 Garantia exactly-once

Coordenar os dois lados para que cada evento seja processado exatamente uma vez.

- **Trade-off que motivou o descarte:** exigiria coordenação entre plataforma e cliente e ficaria muito mais complexo. At-least-once com `event_id` resolve 99% dos casos, e é o que Stripe e GitHub fazem `[09:25] Diego`. O custo aceito é jogar a responsabilidade de deduplicação para o cliente `[09:25] Sofia`, compensado por documentação destacada no portal do desenvolvedor `[09:26] Marcos`.

### 4.6 Secret global da plataforma

Uma única secret compartilhada com todos os clientes.

- **Trade-off que motivou o descarte:** blast radius. Se vaza uma, vaza tudo `[09:21] Sofia`. O risco é concreto: já houve cliente que vazou secret em log de aplicação dele `[09:22] Diego`. Aceitamos o custo de gerenciar secret por endpoint e um fluxo de rotação em troca de conter o impacto de um vazamento.

---

## 5. Questões em aberto

### 5.1 Rate limiting de saída por cliente

Se um cliente tem 50 pedidos mudando de status em um minuto, a plataforma dispara 50 chamadas contra o endpoint dele `[09:38] Diego`.

**Status: não decidido.** A avaliação foi de que não faz parte do escopo desta fase. A postura acordada é **observar e implementar se virar problema** `[09:39] Diego`, ficando registrado como ponto em aberto `[09:39] Larissa`.

**O que precisa ser respondido antes de decidir:** qual a taxa real de mudança de status por customer em produção, e se algum cliente reclama de volume.

### 5.2 Estratégia de escala do worker além de single-worker

A garantia de ordering por `order_id` depende de haver **um único worker** `[09:12] Diego`. Escalar para múltiplos workers em paralelo quebra essa garantia.

**Status: adiado.** Os caminhos foram identificados (particionar por `order_id`, ou usar lock pessimista), mas classificados como problema do futuro, não de agora `[09:13] Diego`.

**O que precisa ser respondido antes de decidir:** qual volume de eventos por segundo torna o single-worker insuficiente, e se algum cliente passa a exigir ordering sob paralelismo.

### 5.3 Revogação imediata de secret comprometida

O grace period de 24h ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) mantém a secret antiga válida em paralelo `[09:21] Sofia`. Se a rotação for motivada por vazamento confirmado, a secret comprometida continua aceita durante a janela.

**Status: não levantado explicitamente na reunião; identificado na redação deste RFC como consequência da decisão de grace period.** Fica como questão a endereçar na revisão de segurança `[09:46] Sofia`.

**O que precisa ser respondido:** se a revogação imediata deve existir como operação distinta da rotação, e quem tem autoridade para executá-la.

### 5.4 Endurecimento de autorização no CRUD de configuração

Por enquanto o CRUD de configuração de webhook aceita qualquer role autenticada, com a ressalva de que mais para frente isso pode ser endurecido `[09:37] Sofia`.

**Status: decisão consciente de postergar.** O critério de quando endurecer não foi definido.

### 5.5 Política de arquivamento da outbox

Linhas entregues devem ser arquivadas depois de ~30 dias, mas isso foi colocado **explicitamente fora do escopo desta feature** `[09:08] Diego`. A tabela cresce indefinidamente até que essa política exista.

**O que precisa ser respondido:** onde a rotina de arquivamento roda e qual o critério de retenção definitivo.

---

## 6. Impacto e riscos

### 6.1 Impacto no sistema existente

| Área | Impacto | Natureza |
| --- | --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` passa a inserir na outbox dentro da transação existente | **Alteração de código crítico.** Único ponto de mudança em fluxo de produção |
| `prisma/schema.prisma` | Duas tabelas novas (`webhook_outbox`, `webhook_dead_letter`) e a de configuração de webhooks | Aditivo, sem migração destrutiva |
| `src/shared/logger/index.ts` | Configuração de `redact` estendida para cobrir a secret | Alteração pontual, obrigatória por segurança |
| `src/middlewares/error.middleware.ts` | **Nenhuma.** Erros novos estendem `AppError` e são tratados pelo middleware atual `[09:29] Bruno` | Sem impacto |
| Infraestrutura | Uma unidade de execução nova (worker), com pool de conexões próprio | Aditivo |

### 6.2 Riscos principais

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Falha na inserção da outbox derruba mudança de status de pedido | Baixa | Alto: bloqueia operação central da plataforma | A inserção é `INSERT` local sem I/O de rede; cobertura por testes de integração no fluxo de `changeStatus`. O rollback é comportamento desejado `[09:40] Bruno` |
| Worker único cai e nenhum evento é entregue | Média | Alto: feature inteira para de funcionar silenciosamente | Métrica de idade do evento pendente mais antigo, com alerta. Detalhado no FDD |
| Vazamento de secret de webhook | Média (já ocorreu com cliente `[09:22] Diego`) | Alto: permite forjar eventos | Secret por endpoint contém o blast radius; rotação com grace period; `redact` no logger; revisão de segurança dedicada `[09:46] Sofia` |
| Crescimento indefinido da outbox | Alta | Médio: degradação progressiva de performance de leitura | Índices em status e `created_at`, batch pequeno `[09:08] Diego`. Arquivamento está fora de escopo, o que mantém o risco aberto (ver 5.5) |
| Cliente não implementa deduplicação por `X-Event-Id` | Média | Médio: efeito colateral duplicado no sistema do cliente | Documentação destacada no portal do desenvolvedor `[09:26] Marcos` |

### 6.3 Dependências de entrega

- **Revisão de segurança da Sofia** antes do deploy, com no mínimo dois dias úteis reservados, cobrindo especificamente HMAC e geração de secret `[09:46] Sofia`. Está incluída na estimativa de três sprints `[09:47] Larissa`.
- **Documentação no portal do desenvolvedor**, sob responsabilidade do Marcos, cobrindo como integrar via API e o comportamento at-least-once `[09:26] Marcos`, `[09:40] Marcos`.

---

## 7. Fora de escopo desta proposta

Itens levantados na reunião e explicitamente descartados ou adiados:

| Item | Decisão | Origem |
| --- | --- | --- |
| Notificação por email quando o webhook do cliente está falhando | **Descartado desta fase.** Fica para a próxima, depois de medir o impacto | `[09:37] Marcos`, `[09:37] Larissa` |
| Dashboard visual para o cliente acompanhar os webhooks | **Fora de escopo.** Apenas endpoints; painel é projeto separado do time de frontend | `[09:39] Marcos`, `[09:40] Larissa` |
| Rate limiting de saída por cliente | **Não decidido**, observar e decidir depois (ver 5.1) | `[09:38]` a `[09:39]` |
| Arquivamento de linhas entregues da outbox | **Fora do escopo desta feature** | `[09:08] Diego` |
| Webhooks inbound (cliente enviando para a plataforma) | **Fora de escopo.** Os clientes querem receber, não mandar | `[09:02] Marcos` |
| Múltiplos workers em paralelo | **Adiado**, problema do futuro (ver 5.2) | `[09:13] Diego` |

---

## 8. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de webhook |
| [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md) | Retry com backoff exponencial de 5 tentativas e DLQ em tabela separada |
| [ADR-003](./adrs/ADR-003-at-least-once-x-event-id.md) | Garantia at-least-once com deduplicação por `X-Event-Id` no cliente |
| [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period |
| [ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md) | Worker em processo separado consumindo a outbox por polling de 2 segundos |
| [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto no módulo de webhooks |

O detalhamento de implementação (fluxos passo a passo, contratos de endpoint com payloads, matriz de erros `WEBHOOK_*`, observabilidade e integração arquivo por arquivo com o código existente) está no [FDD](./FDD.md).

---

## 9. Pedido de revisão

Pontos em que a revisão dos participantes é especialmente solicitada:

- **Diego:** a modelagem da outbox e a política de retry cobrem os casos operacionais que você levantou? Concorda com a formulação das questões em aberto 5.1 e 5.2?
- **Bruno:** a assinatura proposta para o gancho na transação (`publishWebhookEvent(tx, order, fromStatus, toStatus)` `[09:41] Bruno`) está adequada, e o detalhamento no FDD reflete o que você tem em mente para `order.service.ts`?
- **Sofia:** a questão 5.3 (revogação imediata de secret comprometida) é uma lacuna real na decisão de grace period, ou está coberta de outra forma?
- **Marcos:** os itens de fora de escopo da seção 7 refletem o que foi acordado com os clientes?
