# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

- **Status:** Aceita
- **Data:** 2026-08-01
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Origem:** Reunião técnica de webhooks, `[09:19]` a `[09:23]`, `[09:36]`

## Contexto

A feature expõe eventos com dados de pedidos para endpoints HTTP que estão **fora da infraestrutura da plataforma** `[09:19] Sofia`. Isso cria dois requisitos de segurança que não existem no restante da aplicação, onde toda comunicação é inbound e autenticada por JWT (`src/middlewares/auth.middleware.ts`):

1. O cliente precisa conseguir validar que a requisição veio realmente da plataforma, e não de um terceiro se passando por ela.
2. O cliente precisa conseguir detectar se o payload foi adulterado em trânsito `[09:19] Sofia`.

O modelo de autenticação existente não serve: o JWT da plataforma autentica quem chama a API, não quem é chamado por ela. O fluxo aqui é invertido.

Havia também um risco de blast radius a endereçar. A plataforma já teve um caso real de cliente que vazou secret em log de aplicação dele `[09:22] Diego`.

## Decisão

Adotamos **assinatura HMAC-SHA256 sobre o corpo da requisição**, com **secret única por endpoint de webhook** e **suporte a rotação com grace period de 24 horas** `[09:22] Sofia`.

### Assinatura

- Algoritmo **HMAC-SHA256**, escolhido por ser o padrão de mercado, com biblioteca disponível em qualquer stack de cliente sério `[09:20] Sofia`.
- A assinatura é calculada sobre o corpo do request e enviada no header `X-Signature` `[09:20] Sofia`.
- O cliente recalcula a assinatura com a secret compartilhada e compara `[09:20] Sofia`.
- Acompanha o header `X-Timestamp` com o timestamp do envio, para o cliente conseguir detectar replay attack se quiser `[09:44] Diego`.

### Secret por endpoint

Cada endpoint de webhook cadastrado tem **secret própria**, não uma secret global da plataforma. A razão é blast radius: com secret global, o vazamento de uma comprometeria todas as integrações `[09:21] Sofia`.

A secret é **gerada pela plataforma** e devolvida ao cliente no momento da criação do webhook `[09:31] Marcos`.

### Rotação com grace period

O cliente pode solicitar nova secret via API. Quando rotaciona, a **secret antiga permanece válida por 24 horas em paralelo**, para dar tempo de migrar os sistemas dele. Após esse período, a antiga é invalidada `[09:21] Sofia`.

### TLS obrigatório

A URL do webhook precisa ser `https`. Se o cliente cadastrar `http`, a requisição é recusada com erro de validação. Isso foi classificado na reunião como validação de schema Zod, não decisão arquitetural `[09:23] Sofia`, e está especificada no FDD.

### Autorização do endpoint administrativo

O endpoint de replay da DLQ ([ADR-002](./ADR-002-retry-backoff-dlq.md)) exige **role `ADMIN`**, porque mexer em fila de entrega de notificação não é operação de operador `[09:36] Sofia`. Reaproveita o `requireRole` já existente em `src/middlewares/auth.middleware.ts` `[09:36] Larissa`.

O endpoint precisa **logar quem executou o replay, para auditoria** `[09:36] Sofia`.

O restante do CRUD de configuração de webhook aceita qualquer role autenticada por enquanto, com possibilidade de endurecer mais adiante `[09:37] Sofia`.

## Alternativas Consideradas

### Secret global da plataforma

Uma única secret compartilhada com todos os clientes.

- **Descartada.** Vazamento de uma secret comprometeria todas as integrações simultaneamente `[09:21] Sofia`. O risco é concreto: já houve cliente que vazou secret em log de aplicação `[09:22] Diego`.
- **Trade-off do descarte:** secret global seria trivial de gerenciar, sem necessidade de storage por endpoint nem fluxo de rotação. Perdeu para blast radius: o custo de gerenciamento é baixo comparado ao impacto de um vazamento único derrubar a confiança de toda a base.

### Rotação sem grace period (troca imediata)

Invalidar a secret antiga no instante da rotação.

- **Descartada implicitamente pela decisão de grace period.** Forçaria uma janela de indisponibilidade da integração: entre o cliente pedir a nova secret e terminar de fazer o deploy dela nos sistemas dele, toda requisição falharia na validação de assinatura.
- **Trade-off do descarte:** troca imediata é mais segura em caso de vazamento confirmado, porque encerra a validade da secret comprometida na hora. Perdeu para disponibilidade da integração no caso comum (rotação preventiva). As 24 horas são um teto conhecido e curto de exposição.

### mTLS em vez de HMAC

Autenticação mútua por certificado na camada de transporte.

- **Descartada.** Não foi levantada na reunião, mas é a alternativa padrão de mercado ao HMAC. Exigiria que cada cliente provisionasse e mantivesse certificado, elevando muito a barreira de integração para clientes B2B de porte médio. O critério que guiou a escolha do HMAC-SHA256 foi exatamente o oposto: qualquer cliente sério já tem biblioteca para isso `[09:20] Sofia`.
- **Trade-off do descarte:** mTLS autenticaria também o canal e dispensaria gestão de secret no payload. Perdeu para custo de integração e operação de PKI.

## Consequências

### Positivas

- O cliente consegue verificar origem e integridade de cada evento recebido, sem depender de allowlist de IP nossa.
- Blast radius contido: vazamento de secret afeta apenas um endpoint de um cliente.
- Rotação sem downtime da integração, graças ao grace period de 24h.
- Reaproveita o `requireRole` existente para o endpoint administrativo, sem introduzir mecanismo novo de autorização `[09:36] Larissa`.
- `X-Timestamp` habilita detecção de replay attack no lado do cliente, sem impor essa verificação como obrigatória.

### Negativas

- A plataforma passa a armazenar material secreto por endpoint de webhook. Isso cria uma superfície nova: as secrets precisam ser tratadas como dado sensível em storage, em logs e nas respostas de API. A revisão de segurança dedicada endereça esse ponto (ver Dependências).
- O grace period de 24h significa que, durante essa janela, **duas secrets são válidas simultaneamente** para o mesmo endpoint. Se a rotação foi motivada por vazamento confirmado, a secret comprometida continua aceita por até 24 horas. Mitigação: prever revogação imediata como operação distinta da rotação, registrada como questão em aberto no RFC.
- Validar HMAC é responsabilidade do cliente. Um cliente que não valide a assinatura não ganha nenhuma proteção, e a plataforma não tem como detectar isso.
- A geração de secret e a implementação do HMAC são pontos de falha crítica de segurança. Por isso foi acordada **revisão de segurança dedicada antes do deploy**, com reserva de no mínimo dois dias úteis para a Sofia analisar especificamente HMAC e geração de secret `[09:46] Sofia`.

### Trade-off explícito

Trocamos **simplicidade de gestão de credenciais** (secret global) por **contenção de blast radius** (secret por endpoint + rotação). E trocamos **encerramento imediato de secret comprometida** por **disponibilidade da integração durante a rotação** (grace period de 24h).

## Dependências

- Revisão de segurança da Sofia, com no mínimo dois dias úteis reservados antes do deploy, cobrindo HMAC e geração de secret `[09:46] Sofia`.

## Decisões relacionadas

- [ADR-003 — Entrega at-least-once com X-Event-Id](./ADR-003-at-least-once-x-event-id.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](./ADR-002-retry-backoff-dlq.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./ADR-006-reuso-padroes-existentes.md)
