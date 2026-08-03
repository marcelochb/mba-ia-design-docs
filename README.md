# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs produzido a partir da transcrição de uma reunião técnica e do código de um Order Management System em produção.

> O enunciado original do desafio está preservado em [`docs/DESAFIO.md`](./docs/DESAFIO.md).

---

## Sobre o desafio

O ponto de partida foram dois artefatos e nada mais: a gravação literal de uma reunião de 55 minutos entre cinco pessoas (`TRANSCRICAO.md`) e uma aplicação Node.js e TypeScript funcional, um OMS com módulos de autenticação, usuários, clientes, produtos e pedidos. A reunião decidiu construir um sistema de webhooks de notificação de pedidos, mas nada foi registrado além da conversa. A tarefa foi transformar isso em documentação técnica acionável o suficiente para um time começar a implementar.

O que torna o exercício interessante não é gerar texto, é decidir o que **não** entra. A transcrição mistura decisões fechadas, alternativas descartadas, coisas adiadas para fases futuras, perguntas que ficaram sem resposta e detalhes técnicos secundários, tudo no mesmo fluxo de conversa. Aviso por email foi pedido e recusado. Dashboard foi levantado e mandado para outro time. Rate limiting virou "observar e decidir depois". Trigger de banco, Redis Streams, 3 tentativas de retry e exactly-once foram propostos e descartados, cada um por um motivo específico. Se qualquer um desses aparece como requisito, a documentação está errada, mesmo que o texto esteja bem escrito.

A entrega são seis artefatos que operam em alturas diferentes e não se repetem: PRD (produto), RFC (arquitetura, submetido a revisão), ADRs (cada decisão isolada), FDD (implementação), Tracker (rastreabilidade) e este README. O critério que guiou tudo: **nenhuma informação sem origem identificável na transcrição ou no código**.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (Opus 5)** | Ferramenta principal. Leitura direta do código do repositório, análise da transcrição, produção dos documentos e escrita dos arquivos. A vantagem sobre uma interface de chat foi o acesso ao código real: em vez de colar trechos, o modelo abriu `order.service.ts`, `app-error.ts`, `error.middleware.ts`, `auth.middleware.ts`, `logger/index.ts` e `schema.prisma` e extraiu os pontos de integração a partir do que está escrito lá, não do que a transcrição diz sobre eles |
| **Bash + Python via Claude Code** | Verificação programática. Usado para validar os 94 timestamps do tracker contra `TRANSCRICAO.md` por comparação exata, conferir que todo caminho de arquivo citado existe no repositório e recontar as estatísticas de cobertura. Foi o que pegou dois erros reais, descritos na seção de iterações |

Os prompts de entrevista de PRD e FDD disponibilizados pelo professor foram usados como base estrutural, adaptados de modo de entrevista interativa para modo de extração a partir de fonte fixa, conforme mostrado abaixo.

---

## Workflow adotado

A ordem seguiu a sugestão do enunciado, que se mostrou correta na prática: as decisões formam o esqueleto sobre o qual todo o resto se apoia.

**1. Contextualização com o código antes de qualquer documento.**
Antes de escrever uma linha, mapeamento do que existe: `order.service.ts` (a transação de `changeStatus`), a hierarquia de erro em `shared/errors/`, o `errorMiddleware`, o `requireRole`, o `redact` do Pino, as convenções do `schema.prisma` e o padrão de módulo de `src/modules/orders/`. Sem isso, a seção de integração do FDD e as linhas `CODIGO` do tracker seriam inventadas.

**2. Leitura dirigida da transcrição, separando o que entra do que não entra.**
Antes de produzir, classificação explícita de cada ponto da conversa em quatro baldes: decisão fechada, alternativa descartada, item adiado ou fora de escopo, e detalhe técnico secundário. Esse passo é o que evita que uma ideia mencionada às 09:37 e recusada às 09:37 vire requisito.

**3. ADRs primeiro.** Seis ADRs, um por decisão principal. Cada um com a alternativa real que foi discutida e descartada, e o trade-off explícito que motivou o descarte.

**4. RFC.** Consolidação da proposta sobre as decisões já registradas, com as alternativas e as questões em aberto, que têm lugar natural aqui. Mantido conciso e em nível de arquitetura, com links para os ADRs.

**5. FDD.** O documento mais profundo, construído sobre as decisões já formalizadas: fluxos, contratos, matriz de erros e a seção obrigatória de integração com o sistema existente.

**6. PRD.** Produzido por último entre os grandes documentos. Com RFC, FDD e ADRs prontos, virou consolidação em nível de produto, seguindo o esqueleto do prompt do curso.

**7. Tracker.** Varredura dos documentos prontos, item por item, seguida de **verificação por script** dos timestamps e dos caminhos de arquivo.

**8. README e revisão final** contra a checklist de critérios de aceite.

Organização da interação: um documento por vez, com o contexto do código já carregado, sempre pedindo a citação da origem junto de cada afirmação. Pedir a origem **junto** com o conteúdo, e não depois, é o que impede a racionalização retroativa, quando o modelo escreve algo plausível e só então procura um timestamp que sirva de justificativa.

---

## Prompts customizados

### Prompt 1 — Classificação da transcrição antes de qualquer redação

Este foi o prompt mais importante do processo. Ele não gera documento nenhum, apenas força a separação entre o que entra e o que não entra. Rodá-lo antes de produzir qualquer artefato mudou a qualidade de tudo que veio depois.

```
Você vai analisar TRANSCRICAO.md. NÃO produza nenhum documento ainda.

Percorra a transcrição do início ao fim e classifique CADA ponto técnico
discutido em exatamente um destes quatro baldes:

A) DECISÃO FECHADA — foi decidido e entra na entrega
B) ALTERNATIVA DESCARTADA — foi proposta e recusada, com um motivo
C) ADIADO / FORA DE ESCOPO — foi levantado e empurrado para outra fase
D) DETALHE SECUNDÁRIO — decidido, mas é parâmetro de implementação,
   não decisão arquitetural

Para cada item, produza uma linha:
[balde] | [timestamp + falante] | [o que é] | [se B ou C: qual o motivo
exato do descarte, com o timestamp de quem descartou]

Regras:
- Um item mencionado e depois recusado é B ou C, NUNCA A. Se o Marcos
  pediu e a Larissa disse não, o balde é C, não A.
- Se você não consegue apontar timestamp e falante, o item não existe.
  Não o inclua.
- Não infira decisões a partir do que "faria sentido tecnicamente".
  Só entra o que foi dito.
- Ao final, liste separadamente os pontos que ficaram SEM RESPOSTA na
  reunião. Esses são as questões em aberto do RFC, não requisitos.
```

### Prompt 2 — FDD com integração ancorada no código real

Adaptado do prompt de FDD do curso, com duas mudanças: modo de extração em vez de entrevista, e a exigência de que a seção de integração seja ancorada em arquivos que o modelo tenha aberto de fato.

```
Produza docs/FDD.md a partir de TRANSCRICAO.md e do código deste
repositório, seguindo a estrutura do prompt de FDD do curso.

Antes de escrever a seção "Integração com o sistema existente", ABRA e
LEIA estes arquivos. Não escreva sobre eles de memória nem a partir do
que a transcrição diz sobre eles:
  src/modules/orders/order.service.ts
  src/shared/errors/app-error.ts
  src/shared/errors/http-errors.ts
  src/middlewares/error.middleware.ts
  src/middlewares/auth.middleware.ts
  src/shared/logger/index.ts
  prisma/schema.prisma

Para CADA arquivo, descreva:
- o que já existe nele hoje (com nome real de classe, função ou campo)
- como o módulo de webhooks se integra
- se o arquivo PRECISA ser alterado ou se é reuso sem alteração

Restrições:
- Todo código de erro do módulo usa prefixo WEBHOOK_.
- Mínimo de 4 endpoints com exemplo de request e response e status codes.
- Se um arquivo NÃO existe hoje (ex: src/worker.ts), diga explicitamente
  que é arquivo a criar. Nunca o descreva como existente.
- Cada afirmação sobre uma decisão traz o timestamp da transcrição que
  a sustenta, no formato [hh:mm] Nome.
- Não inclua no escopo nada que esteja no balde C da classificação.
```

### Prompt 3 — Verificação adversarial do tracker

Usado depois de o tracker estar pronto, para tentar quebrá-lo em vez de confirmá-lo.

```
O tracker está em docs/TRACKER.md. Sua tarefa NÃO é elogiá-lo, é
encontrar erros nele.

Execute, por script e não por leitura:
1. Extraia todo [hh:mm] Nome da coluna Localização e confirme, por
   comparação exata, que cada um existe em TRANSCRICAO.md. Liste os que
   não existirem.
2. Extraia todo caminho de arquivo das linhas com Fonte = CODIGO e
   confirme que cada um existe no repositório. Liste os ausentes.
3. Reconte as estatísticas de cobertura e compare com os números que o
   próprio documento afirma. Se divergirem, o documento está errado.
4. Procure itens que aparecem como requisito no PRD ou no FDD mas que na
   transcrição foram descartados ou adiados.

Reporte apenas o que estiver errado, com evidência. Não reporte acertos.
```

---

## Iterações e ajustes

Foram cinco ciclos principais. Os quatro problemas concretos abaixo são os que mudaram o resultado de forma relevante.

### 1. Estatísticas do tracker afirmadas sem verificação

A primeira versão do tracker fechava com um bloco de cobertura declarando 148 itens, 117 com fonte `TRANSCRICAO` (79%) e 31 com fonte `CODIGO`. Números plausíveis, escritos com confiança, e **errados**: nenhum tinha sido contado.

A contagem por script devolveu 192 itens, 162 `TRANSCRICAO` (84%) e 30 `CODIGO` (16%). Também revisei a alegação de "60+ timestamps distintos", que na verificação eram 94.

A correção não foi só trocar os números, foi trocar a natureza da afirmação: o bloco passou a declarar explicitamente **como** cada número foi verificado. Um tracker que existe para combater alucinação e que inventa as próprias estatísticas derrota o próprio propósito.

### 2. Arquivos que a feature vai criar tratados como código existente

A verificação de caminhos acusou dois "ausentes": `src/worker.ts` e `src/modules/webhooks/webhook.worker.ts`. Eles não existem porque são justamente o que a feature vai criar. Mas o critério de aceite é literal: nenhum arquivo citado nos documentos pode ser inexistente no repositório.

Revisão de todas as ocorrências para garantir que cada uma está enquadrada como arquivo **a criar** ("será criado um `src/worker.ts`", "entry-point nova"), nunca como código existente, e confirmação de que nenhuma linha do tracker com fonte `CODIGO` aponta para eles. O tracker passou a registrar essa exceção de forma explícita, em vez de deixar o leitor descobrir sozinho.

### 3. Primeiros ADRs com alternativas genéricas em vez das reais

A primeira geração dos ADRs trouxe seções de "Alternativas Consideradas" preenchidas com o que um modelo sabe sobre o assunto em geral: para o ADR de outbox, alternativas como CDC com Debezium e event sourcing. Tecnicamente corretas, e completamente fora do lugar: ninguém falou nisso na reunião.

As alternativas que importam são as que **foram efetivamente colocadas na mesa e recusadas**, cada uma com o motivo exato de quem recusou. Disparo síncrono, descartado porque a transação já é pesada e um cliente lento travaria outros pedidos `[09:04] Bruno`. Redis Streams, descartado como overengineering para um time pequeno `[09:07] Diego`. Trigger de banco, descartada porque o MySQL não tem `NOTIFY`/`LISTEN` `[09:09] Diego`. Três tentativas de retry, descartadas porque cobririam só 30 minutos e já houve cliente com indisponibilidade de duas horas `[09:16] Diego`.

Foi daí que surgiu o Prompt 1: o problema não estava na geração dos ADRs, estava em não ter feito a classificação antes. A única alternativa mantida sem origem na reunião foi mTLS, no ADR-004, e ela está **rotulada** como não levantada na reunião, em vez de disfarçada de discussão real.

### 4. RFC e FDD dizendo a mesma coisa em profundidades parecidas

A primeira versão do RFC descia a detalhe de payload, headers e códigos de erro. O enunciado é explícito sobre isso: o RFC propõe e abre para revisão, o FDD detalha como construir, e conteúdo duplicado é sinal de que algo está no lugar errado.

O RFC foi reescrito para parar no nível de "o que propomos e por quê", com um diagrama de blocos e ponteiros para os ADRs, mais uma seção final de pedido de revisão dirigida nominalmente a cada participante, que é o que faz dele de fato um documento submetido à equipe. Todo o detalhamento de contrato migrou para o FDD.

### Ajuste menor

Também houve uma correção de escopo no meio do caminho, vinda do próprio uso: as questões em aberto do RFC foram separadas entre as que a reunião deixou explicitamente sem resposta (rate limiting, escala do worker, endurecimento de autorização, arquivamento) e uma que **derivou da redação**, a revogação imediata de secret comprometida durante o grace period de 24h. Essa última está marcada como identificada na redação do RFC, não como discussão da reunião. Misturar as duas categorias seria atribuir ao time uma preocupação que ele não teve.

---

## Como navegar a entrega

```
.
├── README.md                    ← você está aqui: o processo
├── TRANSCRICAO.md               ← fonte primária (não alterada)
└── docs/
    ├── DESAFIO.md               ← enunciado original do desafio
    ├── PRD.md                   ← produto: por quê e o quê
    ├── RFC.md                   ← arquitetura: proposta submetida a revisão
    ├── FDD.md                   ← implementação: como construir
    ├── TRACKER.md               ← rastreabilidade de cada item à origem
    └── adrs/
        ├── README.md            ← índice das decisões
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-retry-backoff-dlq.md
        ├── ADR-003-at-least-once-x-event-id.md
        ├── ADR-004-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-worker-processo-separado-polling.md
        └── ADR-006-reuso-padroes-existentes.md
```

### Ordem sugerida de leitura

Cada documento responde a uma pergunta diferente. A ordem abaixo vai do porquê ao como.

1. **[`docs/PRD.md`](./docs/PRD.md)** — por que a feature existe, para quem, com quais métricas e, principalmente, o que ficou de fora
2. **[`docs/RFC.md`](./docs/RFC.md)** — a proposta técnica em nível de arquitetura, as seis alternativas descartadas e as cinco questões em aberto
3. **[`docs/adrs/`](./docs/adrs/)** — cada decisão isolada, com contexto, alternativa real e trade-off explícito. Comece pelo [índice](./docs/adrs/README.md); se for ler só um, leia o [ADR-001](./docs/adrs/ADR-001-outbox-no-mysql.md), que é a decisão da qual todas as outras dependem
4. **[`docs/FDD.md`](./docs/FDD.md)** — o documento para quem vai codar: fluxos, contratos, matriz de erros `WEBHOOK_*` e a seção de integração com o código existente
5. **[`docs/TRACKER.md`](./docs/TRACKER.md)** — a rede de segurança. Consulte sempre que quiser saber de onde veio uma afirmação específica

**Atalhos por interesse:**

| Se você quer... | Vá direto para |
| --- | --- |
| Entender a decisão central | [ADR-001](./docs/adrs/ADR-001-outbox-no-mysql.md), padrão Outbox no MySQL |
| Saber o que mexe no código existente | [FDD, seção 10](./docs/FDD.md#10-integração-com-o-sistema-existente), com 11 arquivos reais mapeados |
| Ver os contratos de API | [FDD, seção 6](./docs/FDD.md#6-contratos-públicos), 7 endpoints com payload |
| Saber o que **não** foi feito e por quê | [PRD, Escopo](./docs/PRD.md#escopo) e [RFC, seção 7](./docs/RFC.md#7-fora-de-escopo-desta-proposta) |
| Conferir a origem de qualquer afirmação | [TRACKER](./docs/TRACKER.md) |

### Observação sobre o código

A entrega é puramente documental. Nenhum arquivo de `src/`, `prisma/`, `tests/` ou de configuração foi alterado. O código serve como contexto e referência, e é citado nos documentos exatamente como está no repositório.
