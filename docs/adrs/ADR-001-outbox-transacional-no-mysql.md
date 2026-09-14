<a id="adr-001"></a>

# ADR-001 — Outbox transacional no MySQL

## Status

Aceito.

## Contexto

O OMS altera o status, registra o histórico e aplica efeitos de estoque dentro de uma única transação Prisma em `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`. Uma chamada HTTP para o cliente dentro dessa transação aumentaria sua duração e acoplaria o sucesso da mudança de pedido à disponibilidade de um sistema externo ([09:04] Bruno (TRANSCRICAO.md:L32); [09:04] Bruno (TRANSCRICAO.md:L36); [09:06] Diego (TRANSCRICAO.md:L48)).

Também é necessário impedir o estado inconsistente “pedido alterado sem evento registrado”. O MySQL já é a persistência do OMS (`prisma/schema.prisma`); adicionar Redis foi considerado infraestrutura excessiva para o tamanho atual do time ([09:07] Diego (TRANSCRICAO.md:L52)).

## Decisão

Persistir o evento de webhook em uma outbox no MySQL existente, na mesma transação que atualiza o pedido, o histórico de status e o estoque. Se a inserção na outbox falhar, toda a transação deve sofrer rollback ([09:06] Diego (TRANSCRICAO.md:L48); [09:40] Bruno (TRANSCRICAO.md:L238); [09:41] Diego (TRANSCRICAO.md:L244)).

O evento terá UUID, seguindo o padrão atual de identificadores (`prisma/schema.prisma`; [09:51] Larissa (TRANSCRICAO.md:L304)), e armazenará o snapshot JSON renderizado no momento da mudança. Assim, mudanças posteriores no pedido não alteram o conteúdo histórico do evento ([09:51] Larissa (TRANSCRICAO.md:L304); [09:52] Larissa (TRANSCRICAO.md:L310)). Os índices devem permitir localizar pendências por estado e antiguidade ([09:08] Diego (TRANSCRICAO.md:L56)).

## Alternativas Consideradas

- <a id="adr-001-alt-01"></a>[ADR-001-ALT-01] **HTTP síncrono dentro de `changeStatus`:** descartado porque endpoint lento ou indisponível bloquearia a transação ou exigiria decidir incorretamente entre rollback do pedido e perda da notificação ([09:04] Bruno (TRANSCRICAO.md:L32)).
- <a id="adr-001-alt-02"></a>[ADR-001-ALT-02] **Redis Streams ou cluster dedicado:** descartado por exigir infraestrutura adicional e operação desproporcional ao cenário inicial ([09:07] Diego (TRANSCRICAO.md:L52)).
- <a id="adr-001-alt-03"></a>[ADR-001-ALT-03] **Guardar somente `order_id` e renderizar no envio:** descartado porque o payload poderia refletir um estado diferente daquele que originou o evento ([09:52] Larissa (TRANSCRICAO.md:L310)).

## Consequências

### Positivas

- <a id="adr-001-pos-01"></a>[ADR-001-POS-01] A alteração do pedido e o registro do evento tornam-se atômicos.
- <a id="adr-001-pos-02"></a>[ADR-001-POS-02] **Análise derivada/proposta:** O worker pode falhar ou reiniciar sem fazer a API depender de uma chamada externa.
- <a id="adr-001-pos-03"></a>[ADR-001-POS-03] O snapshot preserva o fato ocorrido no instante da transição.
- <a id="adr-001-pos-04"></a>[ADR-001-POS-04] A solução reutiliza MySQL e Prisma já presentes no OMS.

### Negativas

- <a id="adr-001-neg-01"></a>[ADR-001-NEG-01] **Análise derivada/proposta:** A aplicação futura precisará de tabelas, índices, estados e manutenção operacional da outbox.
- <a id="adr-001-neg-02"></a>[ADR-001-NEG-02] A tabela pode crescer; arquivamento ou limpeza de eventos entregues foi adiado e permanece fora de escopo ([09:08] Diego (TRANSCRICAO.md:L56)).
- <a id="adr-001-neg-03"></a>[ADR-001-NEG-03] **Análise derivada/proposta:** Janela de candidatos, claim/locking e recuperação de itens abandonados dependem da aprovação do default FDD-PROP-02 antes de implementar o worker.
