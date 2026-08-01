# ADR-003 — Garantia at-least-once com deduplicação por `X-Event-Id` no cliente

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Sofia (Eng. Segurança), Marcos (PM)
- **Origem:** Reunião técnica de webhooks, `[09:24]` a `[09:26]`

## Contexto

A combinação de outbox com retry ([ADR-001](./ADR-001-outbox-no-mysql.md), [ADR-002](./ADR-002-retry-backoff-dlq.md)) introduz a possibilidade de entrega duplicada. O caso clássico: o worker envia a requisição, o cliente processa com sucesso, mas a resposta HTTP se perde ou estoura o timeout. O worker interpreta como falha e retenta. O cliente recebe o mesmo evento duas vezes.

Isso foi assumido explicitamente na reunião: a garantia é at-least-once, e o cliente tem que estar preparado para receber o mesmo evento mais de uma vez `[09:24] Diego`.

A pergunta que restou foi como o cliente diferencia uma reentrega de um evento novo `[09:25] Bruno`.

Houve uma objeção legítima: essa abordagem joga responsabilidade para o lado do cliente `[09:25] Sofia`.

## Decisão

Adotamos **garantia de entrega at-least-once**, com deduplicação delegada ao cliente através de um identificador único de evento transportado no header `X-Event-Id`.

- O `X-Event-Id` carrega um **UUID gerado no momento em que o evento entra na outbox** `[09:25] Diego`. É único por evento, e permanece o mesmo em todas as retentativas daquele evento.
- O cliente deduplica pelo `event_id` do lado dele `[09:25] Diego`.
- O mesmo `event_id` também vai no corpo do payload, como campo `event_id` `[09:43] Diego`.

A responsabilidade transferida ao cliente é compensada por documentação: o comportamento at-least-once será documentado de forma destacada no portal do desenvolvedor `[09:26] Marcos`.

## Alternativas Consideradas

### Garantia exactly-once

Coordenar os dois lados para assegurar que cada evento é processado uma única vez.

- **Descartada.** Exigiria coordenação entre a plataforma e o cliente (protocolo de confirmação em duas fases, ou identificador de sessão de entrega com estado compartilhado), aumentando muito a complexidade da integração. At-least-once com `event_id` resolve 99% dos casos `[09:25] Diego`.
- **Trade-off do descarte:** exactly-once eliminaria a necessidade de dedup no cliente e removeria uma classe inteira de bug de integração. Perdeu para a complexidade de implementação em ambos os lados e para o custo de integração imposto ao cliente, que teria que implementar o protocolo de confirmação.

### Deduplicação no lado da plataforma

Manter registro de entregas confirmadas e suprimir reenvios no servidor.

- **Descartada implicitamente.** Não resolve o caso que gera a duplicata: quando a resposta se perde, a plataforma não sabe se o cliente processou ou não. Suprimir o reenvio nesse cenário transformaria a garantia em at-most-once, com risco de perda de evento, que é pior para o caso de uso.
- **Trade-off:** dedup no servidor pouparia o cliente de trabalho, mas trocaria "risco de duplicata" por "risco de perda", e perda de notificação de mudança de status é o problema que a feature existe para resolver.

## Consequências

### Positivas

- Nenhum evento é perdido por ambiguidade de resposta. Na dúvida, o sistema reenvia, o que é o comportamento correto para notificação de mudança de status.
- Alinhamento com padrão de mercado. Stripe e GitHub adotam a mesma abordagem `[09:25] Diego`, o que reduz a fricção de integração: clientes que já consomem webhooks desses provedores conhecem o padrão.
- O `event_id` como UUID gerado na inserção da outbox serve simultaneamente como chave primária do evento, identificador de deduplicação e correlação em logs.
- Implementação simples do lado da plataforma: não há estado de confirmação para manter.

### Negativas

- Transfere responsabilidade de deduplicação para o cliente `[09:25] Sofia`. Um cliente que ignore o `X-Event-Id` pode processar o mesmo evento duas vezes e produzir efeito colateral duplicado no sistema dele.
- Exige documentação clara e destacada no portal do desenvolvedor, que passa a ser dependência de entrega da feature `[09:26] Marcos`.
- Não há como a plataforma verificar se o cliente implementou a deduplicação corretamente. A corretude da integração fica fora do nosso controle.

### Trade-off explícito

Trocamos **simplicidade da integração do cliente** por **simplicidade e robustez do lado da plataforma**. A decisão se sustenta em dois pontos: é o padrão de mercado, e a alternativa (exactly-once) tem custo de complexidade desproporcional ao ganho para o caso de uso de notificação de status.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](./ADR-002-retry-backoff-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](./ADR-004-hmac-sha256-secret-por-endpoint.md)
