# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este documento é a referência cruzada do pacote de design docs. Cada item registrado no [PRD](./PRD.md), [RFC](./RFC.md), [FDD](./FDD.md) e nos [ADRs](./adrs/) tem aqui a sua origem: um trecho da transcrição da reunião (`TRANSCRICAO.md`) ou um arquivo do código base.

**Função:** garantir que nenhuma informação da documentação foi inventada. Se uma linha não consegue apontar origem, o item não deveria estar no documento.

**Convenção da coluna Localização:**

- Fonte `TRANSCRICAO`: timestamp e falante, no formato `[hh:mm] Nome`
- Fonte `CODIGO`: caminho real do arquivo no repositório

---

## 1. PRD — Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar endpoint de webhook via POST, com secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Editar (PATCH), remover (DELETE) e listar (GET) os webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos: cliente escolhe quais status quer receber por endpoint | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-03a | `docs/PRD.md` | Requisito Funcional | Filtro aplicado na inserção da outbox; se nenhum webhook quer o status, nem insere | TRANSCRICAO | `[09:34] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Publicar evento na outbox dentro da mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| PRD-FR-04a | `docs/PRD.md` | Restrição | Não pode existir caso de status mudar e evento não sair; falha na outbox provoca rollback | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Worker entrega o evento ao endpoint do cliente por HTTP POST | TRANSCRICAO | `[09:09] Diego` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Reentrega com backoff exponencial, máximo de 5 tentativas | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | DLQ em tabela separada com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-07a | `docs/PRD.md` | Requisito Funcional | Replay manual de DLQ via endpoint admin, recolocando na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Assinar payload com HMAC-SHA256 e enviar assinatura em header | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace period de 24h para a secret antiga | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Entrega at-least-once com identificador de evento para dedup no cliente | TRANSCRICAO | `[09:25] Diego` |

## 2. PRD — Contexto, problema e objetivos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | `docs/PRD.md` | Problema | Clientes fazem polling em GET /orders, deixando a integração lenta e cara | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | `docs/PRD.md` | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | `docs/PRD.md` | Restrição | Escopo é apenas outbound; clientes querem receber, não enviar | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-05 | `docs/PRD.md` | Contexto | Outbound webhook simplifica o problema | TRANSCRICAO | `[09:03] Sofia` |
| PRD-CTX-06 | `docs/PRD.md` | Contexto | Aplicação existente é OMS Node.js + TypeScript com MySQL via Prisma | CODIGO | `package.json` |
| PRD-CTX-07 | `docs/PRD.md` | Contexto | Ausência total de mecanismo de notificação, evento ou fila no código base | CODIGO | `src/routes/index.ts` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | Latência abaixo de 10 segundos é considerada tempo real pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo/Métrica | Teto de polling de 2 segundos atende o requisito de 10 segundos | TRANSCRICAO | `[09:09] Diego` |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo/Métrica | Três de três clientes solicitantes migrados para webhook | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo/Métrica | Janela de retry de ~15 horas sustenta meta de entrega acima de 99% | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo/Métrica | Prazo de três sprints incluindo a revisão de segurança | TRANSCRICAO | `[09:47] Larissa` |

## 3. PRD e RFC — Fora de escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OUT-01 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Notificação por email quando webhook falha, adiada para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| PRD-OUT-01a | `docs/PRD.md` | Fora de escopo | Pedido original de aviso por email após 3 falhas seguidas | TRANSCRICAO | `[09:37] Marcos` |
| PRD-OUT-02 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Dashboard visual do cliente; painel é projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-OUT-02a | `docs/PRD.md` | Fora de escopo | Pergunta original sobre painel para o cliente ver os webhooks dele | TRANSCRICAO | `[09:39] Marcos` |
| PRD-OUT-03 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Rate limiting de saída não entra no escopo, observar e implementar se virar problema | TRANSCRICAO | `[09:39] Diego` |
| PRD-OUT-04 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Arquivamento de linhas entregues da outbox fora do escopo desta feature | TRANSCRICAO | `[09:08] Diego` |
| PRD-OUT-05 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Múltiplos workers em paralelo, adiado como problema do futuro | TRANSCRICAO | `[09:13] Diego` |
| PRD-OUT-06 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Garantia exactly-once descartada em favor de at-least-once | TRANSCRICAO | `[09:25] Diego` |
| PRD-OUT-07 | `docs/PRD.md`, `docs/RFC.md` | Fora de escopo | Endurecimento de autorização no CRUD adiado; por enquanto qualquer role autenticada | TRANSCRICAO | `[09:37] Sofia` |

## 4. PRD — Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-NFR-01 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Timeout de 10 segundos na chamada HTTP do worker | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-02 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Limite de 64KB de payload, com erro em vez de truncamento | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-02a | `docs/PRD.md` | Requisito Não Funcional | Posição a favor de erro, não truncamento, em payload muito grande | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-02b | `docs/PRD.md`, `docs/adrs/README.md` | Classificação | Limite de 64KB classificado como requisito não funcional, não decisão arquitetural | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-03 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | URL do webhook obrigatoriamente https; http recusado por validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-04 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Máximo de 5 tentativas de entrega por evento | TRANSCRICAO | `[09:15] Diego` |
| PRD-NFR-05 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Payload enxuto, sem os items do pedido, para não inflar | TRANSCRICAO | `[09:43] Diego` |
| PRD-NFR-05a | `docs/PRD.md` | Requisito Não Funcional | Confirmação de que payload enxuto é o desejado | TRANSCRICAO | `[09:44] Bruno` |
| PRD-NFR-06 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Identificadores em UUID, seguindo o padrão do resto do projeto | TRANSCRICAO | `[09:51] Larissa` |
| PRD-NFR-06a | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Padrão de UUID confirmado no schema existente | CODIGO | `prisma/schema.prisma` |
| PRD-NFR-07 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Logs estruturados no logger existente, sem introduzir ferramenta nova | TRANSCRICAO | `[09:29] Bruno` |
| PRD-NFR-08 | `docs/PRD.md`, `docs/FDD.md` | Restrição | Ordering garantido apenas por order_id e sob single-worker, não global | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-08a | `docs/PRD.md` | Contexto | Clientes nunca pediram garantia de ordering global | TRANSCRICAO | `[09:14] Marcos` |
| PRD-NFR-09 | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Payload é snapshot renderizado na inserção, imune a alterações posteriores | TRANSCRICAO | `[09:52] Larissa` |
| PRD-NFR-09a | `docs/PRD.md`, `docs/FDD.md` | Requisito Não Funcional | Concordância com snapshot na inserção | TRANSCRICAO | `[09:52] Diego` |

## 5. PRD — Riscos e dependências

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RSK-01 | `docs/PRD.md`, `docs/FDD.md`, `docs/RFC.md` | Risco | Worker único é single point of failure; queda interrompe toda a entrega | TRANSCRICAO | `[09:11] Diego` |
| PRD-RSK-02 | `docs/PRD.md`, `docs/FDD.md` | Risco | Falha na inserção da outbox derruba a mudança de status do pedido | TRANSCRICAO | `[09:41] Diego` |
| PRD-RSK-03 | `docs/PRD.md`, `docs/FDD.md` | Risco | Vazamento de secret; já houve cliente que vazou secret em log de aplicação | TRANSCRICAO | `[09:22] Diego` |
| PRD-RSK-04 | `docs/PRD.md`, `docs/FDD.md` | Risco | Crescimento indefinido da outbox por ausência de política de arquivamento | TRANSCRICAO | `[09:08] Diego` |
| PRD-RSK-05 | `docs/PRD.md`, `docs/FDD.md` | Risco | Cliente pode não implementar dedup; responsabilidade transferida a ele | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RSK-06 | `docs/PRD.md`, `docs/RFC.md` | Risco | Volume de chamadas incomoda o cliente: 50 pedidos mudando geram 50 chamadas | TRANSCRICAO | `[09:38] Diego` |
| PRD-DEP-01 | `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` | Dependência | Revisão de segurança com no mínimo dois dias úteis antes do deploy, sobre HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-01a | `docs/PRD.md` | Dependência | Reforço da revisão de segurança antes de subir | TRANSCRICAO | `[09:49] Sofia` |
| PRD-DEP-02 | `docs/PRD.md`, `docs/RFC.md` | Dependência | Documentação no portal do desenvolvedor sobre como integrar via API | TRANSCRICAO | `[09:40] Marcos` |
| PRD-DEP-02a | `docs/PRD.md`, `docs/RFC.md` | Dependência | Documentação destacada do comportamento at-least-once no portal | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Confirmação de prazo com os clientes; Atlas pediu para fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Sessão de revisão do design com Bruno e Diego antes de começar a codar | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-05 | `docs/PRD.md`, `docs/FDD.md` | Dependência | Worker conecta no mesmo banco e usa o mesmo Prisma | TRANSCRICAO | `[09:11] Bruno` |
| PRD-EST-01 | `docs/PRD.md` | Restrição | Estimativa decomposta: outbox/DLQ, worker/retry, CRUD, integração e HMAC | TRANSCRICAO | `[09:46] Larissa` |

## 6. RFC — Alternativas consideradas e descartadas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-ALT-01 | `docs/RFC.md`, `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Disparo síncrono descartado: transação já é pesada, cliente lento travaria outros pedidos | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-01a | `docs/RFC.md` | Trade-off | Sem tratamento razoável para cliente fora do ar; rollback de status não é opção | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-01b | `docs/RFC.md` | Decisão | Síncrono classificado como fora de questão | TRANSCRICAO | `[09:06] Diego` |
| RFC-ALT-02 | `docs/RFC.md`, `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Redis Streams descartado: exigiria subir mais infraestrutura | TRANSCRICAO | `[09:07] Larissa` |
| RFC-ALT-02a | `docs/RFC.md` | Trade-off | Redis Cluster é overengineering para time pequeno; outbox no MySQL resolve | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md`, `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Trade-off | Trigger de banco descartada: MySQL não tem NOTIFY/LISTEN e trigger não notifica processo externo | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-03a | `docs/RFC.md` | Alternativa | Pergunta original sobre usar trigger do banco para ser mais reativo | TRANSCRICAO | `[09:09] Bruno` |
| RFC-ALT-04 | `docs/RFC.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md` | Trade-off | 3 tentativas descartadas: cobriria só ~30 min e mataria evento antes do cliente voltar | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-04a | `docs/RFC.md` | Alternativa | Proposta original de 3 tentativas por ser mais agressivo | TRANSCRICAO | `[09:16] Bruno` |
| RFC-ALT-04b | `docs/RFC.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md` | Trade-off | Retry indefinido descartado: evento ficaria pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-05 | `docs/RFC.md`, `docs/adrs/ADR-003-at-least-once-x-event-id.md` | Trade-off | Exactly-once descartado: exigiria coordenação dos dois lados, muito mais complexo | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-06 | `docs/RFC.md`, `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Trade-off | Secret global descartada: se vaza uma, vaza tudo | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-07 | `docs/RFC.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md` | Trade-off | Status failed na própria outbox descartado em favor de DLQ separada, mais limpa para leitura | TRANSCRICAO | `[09:18] Diego` |

## 7. RFC — Questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída por cliente: observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| RFC-OPEN-01a | `docs/RFC.md` | Questão em aberto | Registrar rate limiting como ponto em aberto, fora do escopo atual | TRANSCRICAO | `[09:39] Diego` |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Estratégia de escala do worker: particionar por order_id ou lock pessimista, adiado | TRANSCRICAO | `[09:13] Diego` |
| RFC-OPEN-02a | `docs/RFC.md` | Questão em aberto | Pergunta original sobre como escalar no futuro | TRANSCRICAO | `[09:13] Bruno` |
| RFC-OPEN-03 | `docs/RFC.md`, `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Questão em aberto | Revogação imediata de secret comprometida; derivada do grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Critério para endurecer autorização do CRUD não definido | TRANSCRICAO | `[09:37] Sofia` |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em aberto | Política de arquivamento da outbox sem definição de onde roda e retenção final | TRANSCRICAO | `[09:08] Diego` |

## 8. FDD — Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks cria endpoint; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id edita o cadastro | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | GET /webhooks lista os webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries retorna os últimos 100 envios com payload, response e tempo | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay reprocessa item da DLQ | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id remove o cadastro | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /webhooks/:id/rotate-secret gera nova secret com grace period | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Formato de resposta paginada { data, pagination } reaproveitado dos módulos existentes | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | customerId vem do body ou do path, não do JWT, porque o JWT é do usuário operador | TRANSCRICAO | `[09:32] Larissa` |
| FDD-CONTRATO-09a | `docs/FDD.md` | Contrato | Observação de que o JWT atual é do operador, não do cliente | TRANSCRICAO | `[09:32] Bruno` |
| FDD-CONTRATO-09b | `docs/FDD.md` | Contrato | Cliente cadastra pela nossa API direto, autenticado com JWT do nosso sistema | TRANSCRICAO | `[09:32] Marcos` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | Tabela de configuração armazena url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` |

## 9. FDD — Payload e headers de saída

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-PAYLOAD-01 | `docs/FDD.md` | Contrato | Payload com event_id, event_type, timestamp ISO, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-02 | `docs/FDD.md` | Contrato | event_type é "order.status_changed" | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-03 | `docs/FDD.md` | Contrato | Sem items no payload; cliente consulta GET /orders/:id se quiser detalhe | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-04 | `docs/FDD.md` | Contrato | Campos order_number e total_cents existem no model Order | CODIGO | `prisma/schema.prisma` |
| FDD-HEADER-01 | `docs/FDD.md`, `docs/adrs/ADR-003-at-least-once-x-event-id.md` | Contrato | X-Event-Id com UUID do evento, gerado na entrada na outbox | TRANSCRICAO | `[09:25] Diego` |
| FDD-HEADER-02 | `docs/FDD.md` | Contrato | X-Signature com o HMAC do payload | TRANSCRICAO | `[09:44] Diego` |
| FDD-HEADER-03 | `docs/FDD.md` | Contrato | X-Timestamp com o timestamp do envio, para o cliente detectar replay attack | TRANSCRICAO | `[09:44] Diego` |
| FDD-HEADER-04 | `docs/FDD.md` | Contrato | X-Webhook-Id com o id do endpoint, para cliente com vários cadastros | TRANSCRICAO | `[09:44] Sofia` |
| FDD-HEADER-05 | `docs/FDD.md` | Contrato | Content-Type application/json | TRANSCRICAO | `[09:44] Diego` |

## 10. FDD — Fluxos e resiliência

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox na mesma transação que atualiza orders e order_status_history | TRANSCRICAO | `[09:06] Diego` |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Transação atual de changeStatus atualiza order, insere no history e ajusta estoque | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Worker lê pendentes em batch pequeno, processa e marca como entregue | TRANSCRICAO | `[09:08] Diego` |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Polling em loop a cada 2 segundos buscando os pendentes mais antigos | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Ordem de processamento por created_at do outbox garante ordem no cliente | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLUXO-06 | `docs/FDD.md`, `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Progressão de backoff 1m, 5m, 30m, 2h, 12h, quase 15 horas no total | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLUXO-06a | `docs/FDD.md` | Trade-off | Cliente fora por 15 horas já tem problema sério; janela considerada aceitável | TRANSCRICAO | `[09:17] Marcos` |
| FDD-FLUXO-07 | `docs/FDD.md` | Fluxo | DLQ com payload, motivo da falha e timestamp, como evidence para debug e reprocessamento | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-08 | `docs/FDD.md` | Fluxo | Replay recoloca o item na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-09 | `docs/FDD.md` | Fluxo | Rotação: secret antiga válida por 24h em paralelo, depois morre | TRANSCRICAO | `[09:21] Sofia` |
| FDD-FLUXO-10 | `docs/FDD.md` | Fluxo | Índice em status e created_at sustenta a query de polling | TRANSCRICAO | `[09:08] Diego` |
| FDD-FLUXO-11 | `docs/FDD.md` | Restrição | Estados do evento na outbox: pendente, processando, falhou, entregue | TRANSCRICAO | `[09:08] Diego` |
| FDD-RESIL-01 | `docs/FDD.md` | Estratégia de resiliência | Timeout de 10s; cliente que não responde é tratado como falha e vai para retry | TRANSCRICAO | `[09:42] Diego` |
| FDD-RESIL-02 | `docs/FDD.md` | Estratégia de resiliência | Worker em processo separado; se a API reinicia, não perde o worker | TRANSCRICAO | `[09:11] Diego` |
| FDD-RESIL-03 | `docs/FDD.md` | Invariante | X-Event-Id estável entre retentativas viabiliza dedup no cliente | TRANSCRICAO | `[09:25] Diego` |
| FDD-RESIL-04 | `docs/FDD.md` | Fallback | Sem fallback de canal: email fora desta fase, único caminho é DLQ e replay manual | TRANSCRICAO | `[09:37] Larissa` |

## 11. FDD — Matriz de erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | `docs/FDD.md` | Restrição | Prefixo WEBHOOK_ obrigatório para todos os códigos do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-02 | `docs/FDD.md` | Erro previsto | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED citados na reunião | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-03 | `docs/FDD.md` | Padrão | Padrão de código de erro existente: INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-04 | `docs/FDD.md` | Padrão | Classe base AppError com statusCode, errorCode e details | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-ERRO-05 | `docs/FDD.md` | Erro previsto | WEBHOOK_PAYLOAD_TOO_LARGE quando o payload excede 64KB | TRANSCRICAO | `[09:24] Diego` |
| FDD-ERRO-06 | `docs/FDD.md` | Erro previsto | WEBHOOK_INVALID_URL quando a URL não é https | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-07 | `docs/FDD.md` | Restrição | Middleware de erro centralizado já trata AppError, Zod e Prisma, sem precisar mudar nada | TRANSCRICAO | `[09:29] Bruno` |
| FDD-ERRO-08 | `docs/FDD.md` | Padrão | Serialização de erro no formato { error: { code, message, details } } | CODIGO | `src/middlewares/error.middleware.ts` |

## 12. FDD — Observabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBS-01 | `docs/FDD.md` | Requisito Não Funcional | Logger Pino já está no projeto inteiro; nada novo é introduzido | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | `docs/FDD.md` | Requisito Não Funcional | Configuração de redact existente cobre token, password e authorization | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | `docs/FDD.md` | Requisito Não Funcional | Redact estendido para secret, porque já houve cliente que vazou secret em log | TRANSCRICAO | `[09:22] Diego` |
| FDD-OBS-04 | `docs/FDD.md` | Requisito Não Funcional | Log de auditoria do replay com quem executou a operação | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-05 | `docs/FDD.md` | Métrica | Métrica de idade do evento pendente mais antigo, para detectar worker parado | TRANSCRICAO | `[09:11] Diego` |
| FDD-OBS-06 | `docs/FDD.md` | Métrica | Histórico de entregas com tempo de resposta, base da métrica de duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-OBS-07 | `docs/FDD.md` | Observabilidade | requestId já produzido pelo request logger serve de base para correlação | CODIGO | `src/middlewares/request-logger.middleware.ts` |

## 13. FDD — Integração com o sistema existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INT-01 | `docs/FDD.md` | Decisão | Alteração crítica no método changeStatus do service de orders | TRANSCRICAO | `[09:40] Bruno` |
| FDD-INT-01a | `docs/FDD.md` | Integração | Método changeStatus com $transaction, update, history e ajuste de estoque | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx da transação | TRANSCRICAO | `[09:41] Bruno` |
| FDD-INT-02a | `docs/FDD.md` | Trade-off | Função pura recebendo o tx, sem injetar repository inteiro | TRANSCRICAO | `[09:41] Diego` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Reuso das classes de erro existentes, estendendo AppError | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Reuso do error middleware centralizado, sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Reuso de authenticate e requireRole para autenticação e autorização | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Reuso do logger Pino, com extensão do redact | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Novas tabelas seguindo convenções de uuid, @@map e @@index do schema | CODIGO | `prisma/schema.prisma` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Worker usa createPrismaClient() para instância própria de PrismaClient | CODIGO | `src/config/database.ts` |
| FDD-INT-08a | `docs/FDD.md` | Decisão | PrismaClient separado por processo, mesmo banco e mesma DATABASE_URL | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Registro do router de webhooks em buildApiRouter | CODIGO | `src/routes/index.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Wiring do controller em buildControllers e prefixo /api/v1 | CODIGO | `src/app.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Enum OrderStatus reaproveitado para validar o filtro de status assinados | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Padrão de rotas com authenticate e validate por rota | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-INT-13 | `docs/FDD.md` | Integração | Padrão de schemas Zod com tipos inferidos via z.infer | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-INT-14 | `docs/FDD.md` | Integração | Middleware validate converte ZodError em ValidationError | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-15 | `docs/FDD.md` | Integração | Testes seguem o padrão existente, com factories e setup atuais | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-16 | `docs/FDD.md` | Integração | Entry-point do worker espelhando o padrão de bootstrap e shutdown do server | CODIGO | `src/server.ts` |
| FDD-INT-16a | `docs/FDD.md` | Decisão | Criar src/worker.ts como entry-point nova, com script npm run worker | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-17 | `docs/FDD.md` | Integração | Configuração nova validada por schema Zod em env | CODIGO | `src/config/env.ts` |
| FDD-INT-18 | `docs/FDD.md` | Integração | Scripts npm seguindo o padrão de dev e start existentes | CODIGO | `package.json` |
| FDD-INT-19 | `docs/FDD.md` | Decisão | Módulo em src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-20 | `docs/FDD.md` | Decisão | Lógica de processamento em webhook.worker.ts ou webhook.processor.ts dentro do módulo | TRANSCRICAO | `[09:28] Bruno` |

## 14. ADRs — Decisões registradas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Padrão Outbox no MySQL, com inserção na mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| ADR-001a | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Fechamento da decisão de outbox em MySQL | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001b | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | Pergunta inicial sobre síncrono no service versus fila/outbox | TRANSCRICAO | `[09:03] Larissa` |
| ADR-001c | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto | Preocupação com performance se acumular muito evento na tabela | TRANSCRICAO | `[09:07] Bruno` |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Backoff exponencial com teto de tentativas e DLQ ao esgotar | TRANSCRICAO | `[09:15] Diego` |
| ADR-002a | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Fechamento: 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Larissa` |
| ADR-002b | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Contexto | Caso real de cliente com indisponibilidade de duas horas em manutenção planejada | TRANSCRICAO | `[09:16] Diego` |
| ADR-002c | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Registro do endpoint admin para replay manual de DLQ | TRANSCRICAO | `[09:19] Larissa` |
| ADR-003 | `docs/adrs/ADR-003-at-least-once-x-event-id.md` | Decisão | Garantia at-least-once; cliente precisa estar preparado para duplicata | TRANSCRICAO | `[09:24] Diego` |
| ADR-003a | `docs/adrs/ADR-003-at-least-once-x-event-id.md` | Decisão | Fechamento: at-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | `[09:26] Larissa` |
| ADR-003b | `docs/adrs/ADR-003-at-least-once-x-event-id.md` | Trade-off | Padrão de mercado: Stripe e GitHub adotam a mesma abordagem | TRANSCRICAO | `[09:25] Diego` |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004a | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Contexto | Cliente precisa validar origem e integridade do payload | TRANSCRICAO | `[09:19] Sofia` |
| ADR-004b | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | SHA-256 escolhido por ser padrão de mercado com biblioteca em todo cliente sério | TRANSCRICAO | `[09:20] Sofia` |
| ADR-004c | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | Role ADMIN obrigatório no replay, reaproveitando o requireRole existente | TRANSCRICAO | `[09:36] Larissa` |
| ADR-004d | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | Mexer em fila de entrega não é coisa de operador; replay exige ADMIN e log de auditoria | TRANSCRICAO | `[09:36] Sofia` |
| ADR-005 | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Worker em polling de 2 segundos; latência mínima de 2s aceita | TRANSCRICAO | `[09:10] Larissa` |
| ADR-005a | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Polling de 2 segundos validado pelo PM contra o requisito de 10 segundos | TRANSCRICAO | `[09:10] Marcos` |
| ADR-005b | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Worker deve rodar em processo separado da instância da API | TRANSCRICAO | `[09:11] Diego` |
| ADR-005c | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Restrição | Mesmo banco e mesma stack, mas não o mesmo processo | TRANSCRICAO | `[09:11] Diego` |
| ADR-005d | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Contexto | Pergunta sobre ordering em mudanças rápidas de PAID, PROCESSING, SHIPPED | TRANSCRICAO | `[09:12] Larissa` |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Reuso máximo: AppError, Pino, error middleware, padrão de módulos, schemas Zod e códigos de erro | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006a | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Estrutura de módulo em src/modules com controller, service, repository, routes e schemas | TRANSCRICAO | `[09:27] Bruno` |
| ADR-006b | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Padrão | Estrutura modular confirmada no código, tomando orders como referência | CODIGO | `src/modules/orders/order.controller.ts` |
| ADR-006c | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Padrão | Repository por módulo como padrão estabelecido | CODIGO | `src/modules/orders/order.repository.ts` |
| ADR-006d | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Códigos de erro seguindo o padrão existente com prefixo WEBHOOK_ | TRANSCRICAO | `[09:28] Bruno` |

## 15. Resumo de fechamento da reunião

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RES-01 | `docs/RFC.md`, `docs/PRD.md` | Decisão | Resumo consolidado de todas as decisões, confirmado pelos participantes | TRANSCRICAO | `[09:48] Larissa` |
| RES-02 | `docs/RFC.md` | Decisão | Confirmação do fechamento pelo engenheiro de plataforma | TRANSCRICAO | `[09:49] Diego` |
| RES-03 | `docs/RFC.md` | Decisão | Confirmação do fechamento pelo engenheiro de pedidos | TRANSCRICAO | `[09:49] Bruno` |
| RES-04 | `docs/PRD.md` | Dependência | PM atualiza os clientes sobre a decisão | TRANSCRICAO | `[09:49] Marcos` |
| RES-05 | `docs/FDD.md`, `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Confirmação de snapshot do payload na inserção | TRANSCRICAO | `[09:52] Bruno` |
| RES-06 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Confirmação de UUID como id da outbox | TRANSCRICAO | `[09:51] Diego` |

---

## Estatísticas de cobertura

| Métrica | Valor |
| --- | --- |
| Total de itens rastreados | 192 |
| Itens com fonte `TRANSCRICAO` | 162 (84%) |
| Itens com fonte `CODIGO` | 30 (16%) |
| Timestamps distintos citados no tracker | 94 |
| Arquivos de código distintos citados no pacote | 25 |
| Falantes citados | 5 de 5 (Larissa, Marcos, Bruno, Diego, Sofia) |

Estes números foram verificados por script contra `TRANSCRICAO.md` e contra a árvore de arquivos do repositório:

- Os 94 timestamps distintos da coluna Localização com fonte `TRANSCRICAO` correspondem a falas literais de `TRANSCRICAO.md`, conferidos por comparação exata de `[hh:mm] Nome`.
- Todos os caminhos da coluna Localização com fonte `CODIGO` existem no repositório.
- Os únicos caminhos citados no pacote que ainda não existem são `src/worker.ts` e `src/modules/webhooks/webhook.worker.ts`, sempre descritos como **arquivos a criar** pela feature, nunca referenciados como código existente nem usados como fonte `CODIGO` neste tracker.

## Itens deliberadamente ausentes da documentação

Estes pontos aparecem na transcrição mas **não** foram registrados como requisito, decisão ou restrição, porque foram descartados, adiados ou classificados como fora de escopo. Estão listados aqui para deixar explícito que a ausência é intencional, não omissão.

| Item | Motivo da ausência | Localização |
| --- | --- | --- |
| Aviso por email ao cliente quando o webhook falha | Recusado nesta fase, adiado para a próxima | `[09:37] Larissa` |
| Dashboard visual para o cliente | Fora de escopo, projeto separado de frontend | `[09:40] Larissa` |
| Rate limiting de saída | Não decidido, apenas registrado como ponto em aberto no RFC | `[09:39] Diego` |
| Arquivamento de linhas entregues em 30 dias | Explicitamente fora do escopo desta feature | `[09:08] Diego` |
| Trigger de banco para consumo reativo | Descartado por limitação do MySQL; consta apenas como alternativa | `[09:09] Diego` |
| Redis Streams | Descartado como overengineering; consta apenas como alternativa | `[09:07] Diego` |
| Retry indefinido e opção de 3 tentativas | Descartados; constam apenas como alternativas | `[09:15] Diego`, `[09:16] Bruno` |
| Particionamento por order_id e lock pessimista | Adiados como problema do futuro; constam como questão em aberto | `[09:13] Diego` |
| Exactly-once | Descartado; consta apenas como alternativa | `[09:25] Diego` |
| Webhooks inbound | Fora de escopo desde a delimitação inicial | `[09:02] Marcos` |
