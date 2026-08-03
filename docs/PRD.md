# PRD: OMS — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-01
Responsável: Marcos (Product Manager)

Documentos relacionados: [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md)

---

### Resumo

O Order Management System (OMS) hoje não oferece nenhuma forma de notificação externa. Clientes B2B que precisam acompanhar o ciclo de vida dos pedidos deles fazem polling em `GET /orders` de tempos em tempos, o que deixa a integração deles lenta e cara `[09:00] Marcos`.

Esta feature entrega um **sistema de webhooks outbound**: quando o status de um pedido muda, a plataforma notifica ativamente os endpoints HTTP que o cliente cadastrou. Os clientes deixam de fazer polling e passam a receber os eventos em segundos.

A entrega inclui o mecanismo de publicação de eventos (padrão Outbox transacional no MySQL existente), um worker de entrega em processo separado com política de retry e Dead Letter Queue, autenticação das requisições por HMAC-SHA256 com secret por endpoint, e os endpoints de API para o cliente configurar seus webhooks e consultar o histórico de entregas.

Nada de infraestrutura nova é provisionado. A feature roda no sistema existente, reaproveitando MySQL, Prisma, o padrão de módulos e os padrões de erro e log já estabelecidos `[09:30] Larissa`.

---

### Contexto e problema

Público-alvo

- Clientes B2B integrados à API do OMS, nominalmente Atlas Comercial, MaxDistribuição e Nova Cargo, que fizeram o pedido formal `[09:00] Marcos`
- Times de engenharia dos clientes, que constroem e mantêm a integração do lado deles
- Operadores ADMIN da plataforma, responsáveis por reprocessar entregas que falharam em definitivo `[09:36] Sofia`

Cenários de uso chave

- Cliente cadastra um endpoint de webhook e escolhe quais status de pedido quer receber, por exemplo apenas `SHIPPED` e `DELIVERED` `[09:33] Marcos`
- Pedido do cliente muda de `PROCESSING` para `SHIPPED` e o sistema do cliente é notificado em segundos, sem precisar consultar a API `[09:00] Marcos`
- Endpoint do cliente fica indisponível durante uma manutenção planejada e os eventos são reentregues automaticamente quando ele volta `[09:16] Diego`
- Cliente investiga uma notificação que acredita não ter recebido, consultando o histórico de entregas com payload, resposta e tempo de resposta `[09:34] Marcos`
- Operador ADMIN reprocessa manualmente um evento que esgotou as tentativas e caiu na Dead Letter Queue `[09:35] Diego`
- Cliente rotaciona a secret do webhook dele e migra os sistemas sem interromper a integração `[09:21] Sofia`

Onde essa feature será implantada

- **Sistema existente.** A feature entra no OMS atual, aplicação Node.js e TypeScript com MySQL via Prisma. Vai como um módulo novo em `src/modules/webhooks`, seguindo a estrutura dos módulos existentes `[09:27] Bruno`, mais uma entry-point de worker em `src/worker.ts`, à semelhança do `src/server.ts` já existente `[09:11] Larissa`. Nenhuma infraestrutura nova é provisionada `[09:07] Diego`

Problemas priorizados

- **Integração lenta e cara para o cliente (prioridade alta).** Os clientes ficam batendo em `GET /orders` de tempos em tempos para descobrir se algo mudou `[09:00] Marcos`. O custo é duplo: consumo desnecessário de requisições do lado deles e do nosso, e atraso na detecção da mudança, proporcional ao intervalo de polling que cada cliente escolheu
- **Risco comercial concreto de perda de cliente (prioridade alta).** A Atlas Comercial sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre `[09:00] Marcos`
- **Ausência total de mecanismo de notificação na plataforma (prioridade alta).** Não existe evento, fila ou webhook no OMS. Qualquer necessidade futura de notificação externa também está bloqueada por esse vácuo
- **Operação manual do cliente (prioridade média).** Sem notificação ativa, o fluxo do cliente depende de alguém atualizando manualmente para ver se mudou algo `[09:02] Marcos`

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar mudanças de status em tempo quase real, conforme percepção do cliente | Latência entre o commit da mudança de status e a primeira tentativa de entrega | **p95 abaixo de 10 segundos**, com teto de polling de 2 segundos `[09:02] Marcos`, `[09:09] Diego` |
| Garantir que nenhuma mudança de status deixe de gerar evento | Razão entre eventos registrados na outbox e mudanças de status com endpoint ativo interessado | **100%.** É invariante transacional, não meta estatística `[09:40] Bruno` |
| Entregar eventos de forma confiável mesmo com indisponibilidade do cliente | Percentual de eventos entregues com sucesso dentro da janela de 5 tentativas | **Acima de 99%**, apoiado na janela de retry de ~15 horas `[09:17] Diego` |
| Eliminar a necessidade de polling dos clientes solicitantes | Número de clientes B2B solicitantes migrados para webhook | **3 de 3** (Atlas, MaxDistribuição e Nova Cargo) `[09:00] Marcos` |
| Não degradar a operação de pedidos existente | Variação da latência p95 de `PATCH /orders/:id/status` após a feature | **Abaixo de 10% de aumento**, já que a operação acrescentada é um `INSERT` local sem I/O de rede |
| Entregar no prazo acordado com o cliente | Sprints até o deploy, incluindo revisão de segurança | **3 sprints** `[09:47] Larissa` |

---

### Escopo

Incluso

- Cadastro de endpoint de webhook, com secret gerada pela plataforma e devolvida na criação `[09:31] Marcos`
- Edição, remoção e listagem dos webhooks de um customer `[09:33] Bruno`
- Filtro de eventos por status: o cliente escolhe quais status quer receber `[09:33] Marcos`
- Publicação transacional do evento na mudança de status, via padrão Outbox no MySQL `[09:06] Diego`
- Worker de entrega em processo separado, com polling de 2 segundos `[09:09] Diego`, `[09:11] Diego`
- Retry com backoff exponencial de 5 tentativas (1m, 5m, 30m, 2h, 12h) `[09:17] Diego`
- Dead Letter Queue persistida em tabela separada `[09:18] Diego`
- Endpoint administrativo de replay manual da DLQ, restrito a role `ADMIN` `[09:35] Diego`, `[09:36] Sofia`
- Assinatura HMAC-SHA256 das requisições, com secret única por endpoint `[09:20] Sofia`, `[09:21] Sofia`
- Rotação de secret com grace period de 24 horas `[09:21] Sofia`
- Histórico de entregas consultável pelo cliente, com sucesso ou falha, payload, resposta e tempo de resposta `[09:34] Marcos`
- Documentação de integração no portal do desenvolvedor, incluindo o comportamento at-least-once `[09:26] Marcos`, `[09:40] Marcos`

Fora de escopo

- **Notificação por email ao cliente quando o webhook dele está falhando.** Foi pedido explicitamente e recusado nesta fase: fica para a próxima, depois de medir o impacto `[09:37] Marcos`, `[09:37] Larissa`
- **Dashboard visual para o cliente acompanhar os webhooks dele.** Apenas endpoints nesta entrega. Painel é projeto separado do time de frontend `[09:39] Marcos`, `[09:40] Larissa`
- **Rate limiting de saída por cliente.** Levantado como preocupação e deixado como ponto em aberto, para observar e implementar se virar problema `[09:38] Diego`, `[09:39] Diego`
- **Arquivamento das linhas entregues da outbox.** Reconhecido como necessário em ~30 dias, mas colocado explicitamente fora do escopo desta feature `[09:08] Diego`
- **Webhooks inbound**, isto é, cliente enviando eventos para a plataforma. Os clientes querem receber, não mandar `[09:02] Marcos`, `[09:03] Sofia`
- **Múltiplos workers em paralelo e ordering global.** Adiado como problema do futuro `[09:13] Diego`. A garantia oferecida é ordering por `order_id` sob single-worker `[09:13] Larissa`
- **Garantia exactly-once.** A entrega é at-least-once, com deduplicação pelo cliente `[09:24] Diego`
- **Endurecimento da autorização no CRUD de configuração.** Por enquanto qualquer role autenticada, com possibilidade de endurecer mais adiante `[09:37] Sofia`

---

### Requisitos funcionais

#### RF-01 Cadastrar endpoint de webhook

O cliente registra uma URL que passará a receber notificações de mudança de status, informando quais status quer ouvir. A secret de assinatura é gerada pela plataforma e devolvida na criação `[09:31] Marcos`.

**Fluxo principal**

- Cliente autenticado chama `POST /webhooks` informando `customerId`, `url` e a lista de status desejados
- O sistema valida que a URL é `https` `[09:23] Sofia`
- O sistema gera uma secret criptograficamente segura, exclusiva daquele endpoint `[09:21] Sofia`
- O sistema persiste o cadastro como ativo
- O sistema retorna `201` com o cadastro e a secret em texto claro, única resposta em que ela é exposta `[09:31] Marcos`

**Fluxos alternativos e exceções**

- `customerId` não corresponde a customer existente, então o cadastro é recusado
- Lista de status contém valor que não pertence ao enum de status de pedido, então o cadastro é recusado
- Customer já tem endpoint ativo com a mesma URL, então o cadastro é recusado como duplicado

**Erros previstos**

- URL com esquema `http` recusada com erro de validação `[09:23] Sofia`
- Filtro de status inválido
- Customer inexistente
- URL duplicada para o mesmo customer

**Prioridade:** alta

---

#### RF-02 Editar, remover e listar webhooks do customer

O cliente mantém a configuração dos endpoints dele ao longo do tempo `[09:33] Bruno`.

**Fluxo principal**

- Cliente chama `PATCH /webhooks/:id` para alterar URL, filtro de status ou estado ativo
- Cliente chama `DELETE /webhooks/:id` para remover um endpoint
- Cliente chama `GET /webhooks?customerId=...` para listar os endpoints dele, em resposta paginada

**Fluxos alternativos e exceções**

- Endpoint inexistente em qualquer das operações, então a operação é recusada
- Endpoint desativado (`active = false`) deixa de receber eventos, sem ser removido

**Erros previstos**

- Webhook não encontrado
- URL inválida na edição
- Identificador em formato inválido

**Prioridade:** alta

---

#### RF-03 Filtrar eventos por status assinado

Cada endpoint escolhe quais status quer receber. O exemplo dado foi "só quero saber quando vira `SHIPPED` e `DELIVERED`" `[09:33] Marcos`.

**Fluxo principal**

- O cliente informa a lista de status no cadastro ou na edição do endpoint
- Quando um pedido muda de status, o sistema seleciona apenas os endpoints ativos do customer que assinaram o status destino
- O filtro é aplicado **no momento da inserção na outbox**. Se nenhum endpoint do customer quer aquele status, nada é inserido, o que economiza linha na tabela `[09:34] Bruno`, `[09:34] Diego`

**Fluxos alternativos e exceções**

- Customer sem nenhum endpoint cadastrado, então nenhum evento é gerado
- Customer com endpoints cadastrados, mas nenhum assinando o status destino, então nenhum evento é gerado
- Customer com múltiplos endpoints interessados, então é gerado um evento por endpoint

**Erros previstos**

- Status informado fora do conjunto de status válidos de pedido

**Prioridade:** alta

---

#### RF-04 Publicar evento de mudança de status de forma transacional

A mudança de status e o registro do evento precisam ser atômicos. Não pode existir caso de status mudar e evento não sair `[09:40] Bruno`.

**Fluxo principal**

- Uma mudança de status de pedido é solicitada
- Dentro da mesma transação que atualiza o pedido, insere no histórico de status e ajusta o estoque, o sistema também insere o evento na outbox `[09:06] Diego`, `[09:40] Bruno`
- O payload do evento é renderizado e gravado no momento da inserção, como snapshot. Se o pedido mudar depois, o evento continua refletindo o estado de quando o status mudou `[09:52] Larissa`
- A transação commita e o evento fica pendente para entrega

**Fluxos alternativos e exceções**

- Qualquer falha na inserção do evento provoca rollback de toda a transação, incluindo a mudança de status `[09:40] Bruno`, `[09:41] Diego`
- Mudança de status sem endpoint interessado não gera evento e a transação segue normalmente

**Erros previstos**

- Payload excedendo o limite de 64KB, que provoca erro e rollback `[09:23] Sofia`, `[09:24] Diego`

**Prioridade:** alta

---

#### RF-05 Entregar o evento ao endpoint do cliente

Um worker em processo separado consome os eventos pendentes e faz as chamadas HTTP `[09:11] Diego`.

**Fluxo principal**

- O worker busca, a cada 2 segundos, os eventos pendentes mais antigos em batch pequeno `[09:09] Diego`, `[09:08] Diego`
- Para cada evento, calcula a assinatura HMAC-SHA256 do corpo com a secret do endpoint `[09:20] Sofia`
- Faz `POST` na URL do endpoint com os headers de identificação e assinatura, respeitando timeout de 10 segundos `[09:42] Diego`, `[09:44] Diego`
- Resposta de sucesso marca o evento como entregue
- O resultado da tentativa é registrado no histórico de entregas `[09:34] Marcos`

**Fluxos alternativos e exceções**

- Endpoint desativado ou removido entre a inserção e o envio, então o evento é encerrado sem tentativa de rede
- Cliente responde depois de 10 segundos, então é tratado como falha `[09:42] Diego`
- Cliente responde com status de erro, então é tratado como falha
- Eventos do mesmo pedido são entregues em ordem de criação, enquanto houver um único worker `[09:12] Diego`

**Erros previstos**

- Timeout da chamada
- Resposta de erro do cliente
- Falha de DNS, TLS ou conexão recusada

**Prioridade:** alta

---

#### RF-06 Reentregar eventos com backoff exponencial

Cliente offline não pode causar perda de evento `[09:14] Larissa`.

**Fluxo principal**

- Em caso de falha, o sistema agenda nova tentativa segundo a progressão 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas `[09:17] Diego`
- São feitas no máximo 5 tentativas, cobrindo uma janela de quase 15 horas `[09:15] Diego`, `[09:17] Diego`
- O identificador do evento permanece o mesmo em todas as tentativas, para permitir a deduplicação pelo cliente `[09:25] Diego`

**Fluxos alternativos e exceções**

- Sucesso em qualquer tentativa encerra o ciclo
- Esgotadas as 5 tentativas, o evento é movido para a Dead Letter Queue `[09:18] Diego`

**Erros previstos**

- Motivo da última falha registrado junto ao evento, para diagnóstico

**Prioridade:** alta

---

#### RF-07 Persistir falhas permanentes em Dead Letter Queue e permitir replay administrativo

Eventos que esgotaram as tentativas ficam registrados como evidência para debug e reprocessamento `[09:18] Diego`.

**Fluxo principal**

- Esgotadas as 5 tentativas, o evento é movido para uma tabela de dead letter separada, com payload, motivo da falha e timestamp `[09:18] Diego`
- Um operador com role `ADMIN` chama `POST /admin/webhooks/dead-letter/:id/replay` `[09:35] Diego`
- O sistema recoloca o evento na outbox como pendente `[09:18] Diego`
- O sistema registra quem executou o replay, para auditoria `[09:36] Sofia`
- O worker entrega o evento no próximo ciclo

**Fluxos alternativos e exceções**

- Usuário sem role `ADMIN` tem a operação recusada `[09:36] Sofia`
- Item de dead letter inexistente, então a operação é recusada
- Item já reprocessado anteriormente, então a operação é recusada

**Erros previstos**

- Falta de permissão
- Item de dead letter não encontrado
- Replay duplicado

**Prioridade:** alta

---

#### RF-08 Autenticar as requisições com HMAC-SHA256

O cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado `[09:19] Sofia`.

**Fluxo principal**

- O worker calcula HMAC-SHA256 sobre o corpo da requisição, usando a secret exclusiva do endpoint `[09:20] Sofia`, `[09:21] Sofia`
- A assinatura é enviada em header dedicado, acompanhada do identificador do evento, do timestamp de envio e do identificador do endpoint `[09:44] Diego`, `[09:44] Sofia`
- O cliente recalcula a assinatura do lado dele e compara `[09:20] Sofia`

**Fluxos alternativos e exceções**

- O timestamp de envio permite ao cliente detectar tentativa de replay attack, se ele quiser implementar essa verificação `[09:44] Diego`
- Cliente com múltiplos endpoints identifica qual cadastro originou o envio pelo identificador do endpoint `[09:44] Sofia`

**Erros previstos**

- Endpoint sem secret vigente tem o envio recusado

**Prioridade:** alta

---

#### RF-09 Rotacionar a secret com grace period

Secret precisa ser rotacionável, e já houve caso de cliente que vazou secret em log de aplicação dele `[09:21] Sofia`, `[09:22] Diego`.

**Fluxo principal**

- O cliente solicita nova secret pela API `[09:21] Sofia`
- O sistema gera a nova secret e a retorna em texto claro
- A secret anterior permanece válida em paralelo por 24 horas, para o cliente ter tempo de migrar os sistemas dele `[09:21] Sofia`
- Após as 24 horas, a secret anterior é invalidada `[09:21] Sofia`

**Fluxos alternativos e exceções**

- Rotação em endpoint inexistente é recusada
- Nova rotação durante a janela de grace period substitui a secret anterior registrada

**Erros previstos**

- Webhook não encontrado

**Prioridade:** alta

---

#### RF-10 Consultar histórico de entregas

O cliente precisa conseguir ver o que foi enviado para ele `[09:34] Marcos`.

**Fluxo principal**

- Cliente chama `GET /webhooks/:id/deliveries`
- O sistema retorna as últimas entregas em resposta paginada, ordenadas da mais recente para a mais antiga, com padrão de 100 registros `[09:34] Marcos`
- Cada registro traz sucesso ou falha, número da tentativa, payload, resposta do cliente e tempo de resposta `[09:34] Marcos`

**Fluxos alternativos e exceções**

- Webhook inexistente, então a consulta é recusada
- Endpoint sem entregas ainda, então a lista retorna vazia

**Erros previstos**

- Webhook não encontrado

**Prioridade:** média

---

#### RF-11 Garantir entrega at-least-once com identificador de evento

O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado `[09:24] Diego`.

**Fluxo principal**

- Cada evento recebe um identificador único no momento em que entra na outbox `[09:25] Diego`
- O identificador é enviado em header e também no corpo do payload `[09:25] Diego`, `[09:43] Diego`
- O cliente deduplica pelo identificador do lado dele `[09:25] Diego`

**Fluxos alternativos e exceções**

- Reentregas do mesmo evento carregam o mesmo identificador `[09:25] Diego`
- Replay administrativo de item da dead letter gera um evento novo, com identificador novo

**Erros previstos**

- Não aplicável. É característica de contrato, documentada de forma destacada no portal do desenvolvedor `[09:26] Marcos`

**Prioridade:** alta

---

### Requisitos não funcionais

Performance

- Latência entre o commit da mudança de status e a primeira tentativa de entrega abaixo de 10 segundos, com teto de polling de 2 segundos `[09:02] Marcos`, `[09:09] Diego`
- Timeout de 10 segundos por chamada ao endpoint do cliente `[09:42] Diego`
- Leitura da outbox em batch pequeno, apoiada em índices de status e data de criação `[09:08] Diego`
- Impacto na latência de `PATCH /orders/:id/status` abaixo de 10%, já que a operação acrescentada é um `INSERT` local

Disponibilidade

- A indisponibilidade do endpoint do cliente não afeta a operação de pedidos da plataforma `[09:04] Bruno`
- A entrega tolera indisponibilidade do cliente por até aproximadamente 15 horas, cobrindo casos reais já observados de manutenção planejada de duas horas `[09:16] Diego`, `[09:17] Diego`
- O worker roda em processo separado, de forma que restart da API não interrompe a entrega `[09:11] Diego`
- **Limitação conhecida:** com um único worker, a queda dele interrompe toda a entrega até o restabelecimento. Nenhum evento é perdido, porque a outbox é durável

Segurança e autorização

- Toda requisição de saída é assinada com HMAC-SHA256 sobre o corpo `[09:20] Sofia`
- Secret única por endpoint, nunca global `[09:21] Sofia`
- Secret rotacionável, com grace period de 24 horas `[09:21] Sofia`
- URL do webhook obrigatoriamente `https`. Cadastro com `http` é recusado `[09:23] Sofia`
- Endpoint de replay da dead letter exige role `ADMIN` `[09:36] Sofia`
- Replay registra em log quem o executou, para auditoria `[09:36] Sofia`
- CRUD de configuração aceita qualquer role autenticada nesta fase, com possibilidade de endurecimento futuro `[09:37] Sofia`
- Secret nunca aparece em log. A configuração de redação do logger é estendida para cobri-la, dado o histórico de vazamento em log de aplicação de cliente `[09:22] Diego`
- Revisão de segurança dedicada antes do deploy, com no mínimo dois dias úteis reservados, cobrindo HMAC e geração de secret `[09:46] Sofia`

Observabilidade

- Logs estruturados no logger existente do projeto, sem introduzir ferramenta nova `[09:29] Bruno`
- Métrica de idade do evento pendente mais antigo, com alerta, para detectar worker parado
- Métricas de volume de eventos, taxa de sucesso de entrega, retentativas e volume de dead letter
- Histórico de entregas persistido e consultável pelo cliente `[09:34] Marcos`
- Identificador do evento serve como chave de correlação de ponta a ponta, desde a publicação até a entrega

Confiabilidade e integridade de dados

- A inserção do evento acontece na mesma transação da mudança de status. Falha na inserção provoca rollback de tudo `[09:40] Bruno`, `[09:41] Diego`
- Payload é snapshot do momento da mudança de status, imune a alterações posteriores no pedido `[09:52] Larissa`
- Todo evento converge para entregue ou para a dead letter. Não existe evento pendurado indefinidamente `[09:15] Diego`
- Ordering garantido por pedido, enquanto houver um único worker. **Não há garantia de ordering global** `[09:13] Larissa`, `[09:12] Diego`
- Entrega at-least-once. Duplicidade é possível e deve ser tratada pelo cliente `[09:24] Diego`

Compatibilidade e portabilidade

- Nenhum contrato de API existente é alterado. `PATCH /orders/:id/status` mantém request e response idênticos
- Migração de banco puramente aditiva, sem alteração destrutiva em tabelas existentes
- Nenhuma dependência de terceiro nova e nenhuma infraestrutura nova. Reaproveita MySQL, Prisma e a stack atual `[09:07] Diego`, `[09:11] Bruno`, `[09:29] Bruno`
- Identificadores em UUID, seguindo o padrão do resto do projeto `[09:51] Larissa`
- **Novo modo de falha:** mudança de status pode falhar por payload acima de 64KB, o que não acontecia antes. É consequência aceita da exigência de atomicidade `[09:24] Larissa`, `[09:41] Diego`

Limites operacionais

- Tamanho máximo de payload de 64KB. Acima disso o sistema retorna erro em vez de truncar, porque payload desse tamanho indica que algo está errado `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`
- Máximo de 5 tentativas de entrega por evento `[09:15] Diego`
- Payload mantido enxuto, sem os itens do pedido. O cliente que precisar de detalhe consulta o pedido pela API `[09:43] Diego`, `[09:44] Bruno`

---

### Arquitetura e abordagem

Abordagem

- Padrão **Outbox transacional** no MySQL já existente: o evento é gravado na mesma transação que muda o status do pedido, e um worker separado faz a entrega `[09:06] Diego`. A escolha se justifica porque disparo síncrono acoplaria a operação de pedidos à disponibilidade do cliente `[09:04] Bruno`, e subir um broker dedicado seria overengineering para um time pequeno `[09:07] Diego`
- O detalhamento da arquitetura está no [RFC](./RFC.md) e a especificação de implementação no [FDD](./FDD.md)

Componentes

- **Módulo de webhooks** em `src/modules/webhooks`, com controller, service, repository, routes e schemas, seguindo a estrutura dos módulos existentes `[09:27] Bruno`
- **Tabela de outbox** no MySQL, com índices em status e data de criação `[09:08] Diego`
- **Tabela de dead letter** separada, com payload, motivo da falha e timestamp `[09:18] Diego`
- **Tabela de configuração de webhook**, com url, secret, customer e estado ativo `[09:21] Bruno`
- **Tabela de histórico de entregas**, alimentando a consulta do cliente `[09:34] Marcos`
- **Worker** em entry-point separada (`src/worker.ts`), com a lógica de processamento dentro do módulo `[09:11] Larissa`, `[09:28] Bruno`

Integrações

- **Mudança de status de pedido:** o serviço de pedidos passa a publicar o evento na outbox, por meio de uma função que recebe o cliente de transação corrente, sem injeção de repository `[09:41] Bruno`, `[09:41] Diego`
- **Endpoints HTTP dos clientes:** comunicação outbound por HTTPS, autenticada por HMAC `[09:20] Sofia`
- **Portal do desenvolvedor:** documentação de integração e do comportamento at-least-once, sob responsabilidade do PM `[09:26] Marcos`, `[09:40] Marcos`

---

### Decisões e trade-offs

#### Decisão: Padrão Outbox no MySQL existente, em vez de disparo síncrono ou broker dedicado

- **Justificativa:** garante atomicidade entre mudança de status e registro do evento sem infraestrutura nova. Disparo síncrono travaria a mudança de status de outros pedidos por causa de um cliente lento e não teria tratamento razoável para cliente fora do ar `[09:04] Bruno`. Broker dedicado seria overengineering para o time `[09:07] Diego`
- **Trade-off:** abre mão do throughput e do fan-out nativo de um broker, e faz a tabela crescer indefinidamente até que exista política de arquivamento, que está fora de escopo `[09:08] Diego`
- Registro completo em [ADR-001](./adrs/ADR-001-outbox-no-mysql.md)

#### Decisão: Worker em processo separado com polling de 2 segundos, single-worker

- **Justificativa:** o MySQL não tem listener nativo para consumo reativo, e polling de 2 segundos atende o requisito de 10 segundos com folga `[09:09] Diego`. Processo separado evita que restart da API derrube o worker `[09:11] Diego`. Single-worker entrega ordering por pedido de graça `[09:12] Diego`
- **Trade-off:** latência mínima de 2 segundos por construção `[09:10] Larissa`, e o worker único é single point of failure. Escalar horizontalmente quebraria o ordering, e o caminho para isso foi adiado `[09:13] Diego`
- Registro completo em [ADR-005](./adrs/ADR-005-worker-processo-separado-polling.md)

#### Decisão: 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada

- **Justificativa:** cobre janelas reais de indisponibilidade de cliente, incluindo o caso já observado de duas horas em manutenção planejada. Três tentativas cobririam apenas 30 minutos e matariam o evento antes do cliente voltar `[09:16] Diego`. Teto de tentativas evita evento pendurado para sempre `[09:15] Diego`
- **Trade-off:** um cliente indisponível pode levar até cerca de 15 horas para receber o evento, aceito explicitamente `[09:17] Marcos`. Reprocessamento a partir da dead letter é manual, exigindo ação humana
- Registro completo em [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md)

#### Decisão: HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Justificativa:** HMAC-SHA256 é padrão de mercado e todo cliente sério tem biblioteca para isso `[09:20] Sofia`. Secret por endpoint contém o blast radius: se vaza uma, não vaza tudo `[09:21] Sofia`. O grace period permite ao cliente migrar sem interromper a integração `[09:21] Sofia`
- **Trade-off:** a plataforma passa a armazenar material secreto por endpoint, criando superfície nova de risco. Durante as 24 horas de grace period, duas secrets são válidas ao mesmo tempo, o que significa que uma secret comprometida continua aceita nessa janela
- Registro completo em [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)

#### Decisão: Entrega at-least-once com deduplicação pelo cliente

- **Justificativa:** garantir exactly-once exigiria coordenação dos dois lados e ficaria muito mais complexo. At-least-once com identificador de evento resolve 99% dos casos e é o que Stripe e GitHub fazem `[09:25] Diego`
- **Trade-off:** joga a responsabilidade de deduplicação para o cliente `[09:25] Sofia`, o que cria dependência de documentação clara no portal do desenvolvedor `[09:26] Marcos` e deixa a corretude da integração fora do controle da plataforma
- Registro completo em [ADR-003](./adrs/ADR-003-at-least-once-x-event-id.md)

#### Decisão: Reuso máximo dos padrões existentes do projeto

- **Justificativa:** o projeto tem padrão claro de módulos, classes de erro, códigos de erro, logger e schemas. Reaproveitar tudo mantém a codebase consistente e faz o middleware de erro existente tratar os erros novos sem alteração `[09:29] Bruno`, `[09:30] Larissa`
- **Trade-off:** o padrão de módulo CRUD existente não tem nada equivalente a um worker de processamento contínuo, então o worker fica dividido entre uma entry-point fora de `src/modules` e a lógica dentro do módulo. Herdar as convenções também significa herdar suas limitações
- Registro completo em [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md)

#### Decisão: Payload enxuto, sem os itens do pedido

- **Justificativa:** mantém o evento pequeno e o custo de armazenamento do snapshot baixo. Cliente que precisar de detalhe consulta o pedido pela API `[09:43] Diego`, `[09:44] Bruno`
- **Trade-off:** cliente que precisa dos itens tem que fazer uma chamada adicional, o que reintroduz parcialmente o padrão de consulta que a feature veio eliminar

---

### Dependências

#### Organizacional: Revisão de segurança antes do deploy

A engenheira de segurança pediu no mínimo dois dias úteis reservados para revisar o código de segurança antes do deploy, olhando com calma especificamente o HMAC e a geração de secret `[09:46] Sofia`. A revisão está incluída na estimativa de três sprints `[09:47] Larissa`, e é bloqueante para a subida.

#### Organizacional: Documentação no portal do desenvolvedor

O PM assumiu documentar no portal como os clientes integram via API `[09:40] Marcos`, incluindo destaque para o comportamento at-least-once e a necessidade de deduplicação pelo identificador do evento `[09:26] Marcos`. Sem isso, RF-11 fica incompleto na prática, porque a corretude da integração depende de o cliente conhecer o contrato.

#### Organizacional: Confirmação de prazo com os clientes

O PM confirma o prazo com os clientes, em especial com a Atlas, que pediu a entrega para o fim de novembro `[09:45] Marcos`, `[09:47] Marcos`.

#### Técnica: Máquina de estados e transação de mudança de status existentes

A publicação do evento depende do fluxo transacional de mudança de status que já existe na aplicação. A alteração crítica acontece exatamente ali `[09:40] Bruno`, o que torna esse código pré-requisito e principal ponto de atenção de regressão.

#### Técnica: Banco MySQL e Prisma já em operação

O worker conecta no mesmo banco, com a mesma configuração de conexão, instanciando um cliente próprio porque é outro processo `[09:11] Bruno`, `[09:30] Bruno`.

#### Técnica: Sessão de revisão do design com o time

A tech lead marcaria uma sessão para o engenheiro de pedidos e o engenheiro de plataforma revisarem o documento de design antes do início da implementação `[09:50] Larissa`. O [RFC](./RFC.md) é o artefato submetido a essa revisão.

---

### Riscos e mitigação

#### Worker único cai e a entrega para silenciosamente

- **Probabilidade:** média
- **Impacto:** alto. A feature inteira para de funcionar e nada no fluxo de pedidos sinaliza o problema. É o single point of failure aceito no desenho, já que o worker é único e separado `[09:11] Diego`, `[09:12] Diego`
- **Mitigação:**
  - Alerta crítico sobre a idade do evento pendente mais antigo
  - Rotina de recuperação que devolve para pendente os eventos travados em processamento
  - Supervisor de processo com restart automático
- **Plano de contingência:** subir o worker manualmente. Como a outbox é durável, nenhum evento é perdido: o backlog é entregue quando o worker volta

#### Falha na publicação do evento derruba a mudança de status do pedido

- **Probabilidade:** baixa
- **Impacto:** alto. Bloqueia a operação central da plataforma, não apenas a notificação. É a contrapartida direta da exigência de atomicidade `[09:41] Diego`
- **Mitigação:**
  - A operação acrescentada é uma escrita local, sem chamada de rede dentro da transação
  - Payload enxuto mantém remoto o risco de estouro do limite de 64KB `[09:43] Diego`
  - Teste de integração cobrindo explicitamente o caminho de rollback
- **Plano de contingência:** desativar os endpoints de webhook do customer afetado, o que faz a publicação retornar sem inserir e restaura o fluxo de pedidos imediatamente

#### Vazamento de secret de webhook

- **Probabilidade:** média. Já aconteceu com um cliente da plataforma, que vazou secret em log de aplicação dele `[09:22] Diego`
- **Impacto:** alto. Permite a um terceiro forjar eventos que o cliente aceitará como legítimos
- **Mitigação:**
  - Secret por endpoint, contendo o blast radius `[09:21] Sofia`
  - Rotação com grace period, sem downtime da integração `[09:21] Sofia`
  - Redação da secret na configuração de log
  - Secret exposta em texto claro apenas na criação e na rotação
  - Revisão de segurança dedicada antes do deploy `[09:46] Sofia`
- **Plano de contingência:** rotação imediata da secret do endpoint afetado. **Limitação conhecida:** o grace period mantém a secret comprometida válida por até 24 horas, e não existe operação de revogação imediata nesta entrega. Registrado como questão em aberto no [RFC](./RFC.md)

#### Crescimento indefinido da tabela de outbox

- **Probabilidade:** alta. É consequência certa do desenho, porque o arquivamento ficou fora de escopo `[09:08] Diego`
- **Impacto:** médio. Degradação progressiva da leitura dos eventos pendentes
- **Mitigação:**
  - Índices em status e data de criação, que mantêm a leitura dos pendentes barata independentemente do volume de entregues `[09:08] Diego`
  - Leitura em batch pequeno
  - Monitoramento do tamanho da tabela
- **Plano de contingência:** arquivamento manual das linhas entregues antigas até que a rotina automática seja implementada em fase futura

#### Cliente não implementa deduplicação e processa evento duplicado

- **Probabilidade:** média
- **Impacto:** médio. Efeito colateral duplicado no sistema do cliente, com potencial de disputa comercial. A responsabilidade foi transferida ao cliente de forma consciente `[09:25] Sofia`
- **Mitigação:**
  - Documentação destacada no portal do desenvolvedor `[09:26] Marcos`
  - Identificador estável entre reentregas, o que torna a deduplicação trivial de implementar `[09:25] Diego`
- **Plano de contingência:** histórico de entregas permite reconstruir com o cliente exatamente o que foi enviado e quantas vezes `[09:34] Marcos`

#### Volume de chamadas incomoda o endpoint do cliente

- **Probabilidade:** média
- **Impacto:** médio. Um cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas `[09:38] Diego`
- **Mitigação:**
  - Postura acordada de observar e implementar rate limiting de saída somente se virar problema `[09:39] Diego`
  - Métrica de duração de entrega por endpoint, para detectar cliente em dificuldade
- **Plano de contingência:** desativar temporariamente o endpoint problemático e priorizar a decisão sobre rate limiting, hoje registrada como ponto em aberto `[09:39] Larissa`

#### Atraso na entrega compromete a relação com o cliente

- **Probabilidade:** baixa
- **Impacto:** alto. A Atlas sinalizou possibilidade de migrar para um concorrente se a entrega não sair no prazo `[09:00] Marcos`
- **Mitigação:**
  - Estimativa decomposta em três sprints, com revisão de segurança já incluída `[09:46] Larissa`, `[09:47] Larissa`
  - Escopo enxuto, com email, dashboard e rate limiting explicitamente fora `[09:37] Larissa`, `[09:40] Larissa`, `[09:39] Diego`
  - Sessão de revisão do design antes do início da implementação `[09:50] Larissa`
- **Plano de contingência:** priorizar a entrega para os três clientes solicitantes, adiando refinamentos não bloqueantes como o histórico de entregas, que é o único requisito de prioridade média

---

### Critérios de aceitação

- Mudança de status de pedido com endpoint ativo interessado gera exatamente um evento por endpoint, registrado na mesma transação da mudança de status `[09:40] Bruno`
- Falha na publicação do evento provoca rollback da mudança de status e do ajuste de estoque, comprovado por teste de integração `[09:41] Diego`
- Mudança para status que nenhum endpoint assinou não gera nenhum evento `[09:34] Bruno`
- Payload do evento é snapshot: alterar o pedido depois não altera o conteúdo do evento já publicado `[09:52] Larissa`
- Evento pendente é entregue em até 2 segundos somados à latência do endpoint do cliente `[09:09] Diego`
- Cliente que não responde em 10 segundos é tratado como falha e o evento é reagendado `[09:42] Diego`
- Falhas seguem a progressão exata de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, no máximo 5 tentativas `[09:17] Diego`
- Esgotadas as 5 tentativas, o evento aparece na dead letter com payload, motivo da falha e total de tentativas `[09:18] Diego`
- Três mudanças de status sequenciais no mesmo pedido chegam ao cliente na ordem correta `[09:12] Diego`
- O identificador do evento é idêntico em todas as reentregas do mesmo evento `[09:25] Diego`
- A assinatura enviada valida como HMAC-SHA256 do corpo com a secret do endpoint, comprovado por teste que recalcula a assinatura `[09:20] Sofia`
- Cada endpoint tem secret distinta, sem nenhuma secret compartilhada entre endpoints `[09:21] Sofia`
- Cadastro de webhook com URL `http` é recusado com erro de validação `[09:23] Sofia`
- Após rotação, requisições assinadas com a secret anterior continuam verificáveis por 24 horas `[09:21] Sofia`
- Secret aparece em texto claro apenas nas respostas de criação e de rotação, nunca em listagem, histórico ou log `[09:22] Diego`
- Replay de dead letter sem role `ADMIN` é recusado `[09:36] Sofia`
- Replay bem-sucedido registra em log quem o executou `[09:36] Sofia`
- Payload acima de 64KB gera erro, sem truncar `[09:23] Sofia`
- Todos os códigos de erro do módulo usam o prefixo padronizado do projeto para webhooks `[09:29] Larissa`
- Erros do módulo são serializados pelo tratamento de erro centralizado existente, sem alteração nele `[09:29] Bruno`
- Payload enviado contém apenas os campos acordados, sem os itens do pedido `[09:43] Diego`
- Histórico de entregas retorna sucesso ou falha, payload, resposta e tempo de resposta `[09:34] Marcos`
- Nenhum contrato de API existente é alterado, e a migração de banco é puramente aditiva
- Revisão de segurança concluída com no mínimo dois dias úteis dedicados antes do deploy `[09:46] Sofia`
- Documentação de integração publicada no portal do desenvolvedor, com destaque para o comportamento at-least-once `[09:26] Marcos`

---

### Testes e validação

Tipos de teste obrigatórios

- **Testes unitários** para as regras críticas: cálculo do HMAC, progressão do backoff, filtro de status assinados e validação do limite de tamanho de payload
- **Testes de integração** para o fluxo principal: transação de mudança de status com publicação na outbox, incluindo o caminho de rollback; ciclo completo do worker contra um servidor HTTP de teste; movimentação para dead letter após esgotar as tentativas; e replay administrativo
- **Testes de contrato** dos endpoints de configuração, histórico e replay, cobrindo os status codes e os códigos de erro especificados
- **Teste de autorização** verificando que o replay exige role `ADMIN` e que o CRUD exige autenticação `[09:36] Sofia`
- **Teste de segurança** verificando que nenhum log contém valor de secret ou assinatura, com a redação do logger ativa `[09:22] Diego`
- **Revisão de segurança manual** conduzida pela engenheira de segurança, com no mínimo dois dias úteis reservados, focada em HMAC e geração de secret `[09:46] Sofia`
- **Teste de ordering** validando que mudanças sequenciais no mesmo pedido chegam em ordem `[09:12] Diego`

Estratégia de validação

- Testes automatizados seguindo o padrão já existente na suíte do projeto, com as factories e o setup atuais
- Validação exploratória do ciclo completo em ambiente de desenvolvimento, com um endpoint receptor de teste que permita simular falha, lentidão dentro e além do timeout, e indisponibilidade prolongada
- Validação do backoff com relógio controlado, sem esperar as 15 horas reais da janela completa
- Revisão de segurança bloqueante antes do deploy, como último passo antes da subida `[09:46] Sofia`, `[09:49] Sofia`
- Validação de integração com um dos clientes solicitantes antes da liberação geral, dado que a corretude da deduplicação depende da implementação do lado dele `[09:25] Sofia`
