<a id="adr-002"></a>

# ADR-002 — Worker separado com polling

## Status

Aceito.

## Contexto

Depois do commit da outbox, um componente precisa localizar eventos pendentes e chamar endpoints externos sem bloquear a API. O MySQL não oferece ao processo Node.js um mecanismo equivalente ao `LISTEN/NOTIFY` do PostgreSQL; triggers executam SQL, mas não notificam diretamente um consumidor externo ([09:09] Diego (TRANSCRICAO.md:L64)).

A expectativa de produto é receber a notificação em menos de 10 segundos ([09:02] Marcos (TRANSCRICAO.md:L22)). O runtime atual já possui um entry point Node.js em `src/server.ts` e uma fábrica de `PrismaClient` em `src/config/database.ts`.

## Decisão

Executar um worker Node.js em processo separado da API, com entry point próprio, mesma `DATABASE_URL` e instância própria de `PrismaClient` por processo ([09:11] Larissa (TRANSCRICAO.md:L72); [09:11] Diego (TRANSCRICAO.md:L70); [09:30] Bruno (TRANSCRICAO.md:L178); [09:30] Larissa (TRANSCRICAO.md:L180)).

Nesta fase haverá um único worker, que fará polling das pendências mais antigas a cada 2 segundos ([09:09] Diego (TRANSCRICAO.md:L60); [09:12] Diego (TRANSCRICAO.md:L80)). Não existe garantia de ordenação global, nem garantia mantida após futura escala horizontal ([09:12] Diego (TRANSCRICAO.md:L80); [09:13] Larissa (TRANSCRICAO.md:L86); [09:13] Diego (TRANSCRICAO.md:L84)).

O single-worker, isoladamente, também não preserva ordem por pedido durante backoff: se o evento anterior ainda não estiver elegível para retry, o posterior pode ser selecionado. Como complemento **proposto, ainda sujeito a aprovação**, FDD-PROP-05 atribui sequência monotônica às transições e impede o claim quando há predecessor não terminal do mesmo (`webhookId`, `orderId`); outro destino permanece independente. `DELIVERED`, DLQ ou cancelamento liberam o seguinte, e replay posterior pode chegar fora de ordem.

## Alternativas Consideradas

- <a id="adr-002-alt-01"></a>[ADR-002-ALT-01] **Worker dentro do processo da API:** descartado para separar ciclo de vida e falhas; reinícios ou escala da API não devem controlar implicitamente o consumidor ([09:11] Diego (TRANSCRICAO.md:L70)).
- <a id="adr-002-alt-02"></a>[ADR-002-ALT-02] **Trigger no MySQL:** descartado porque não entrega uma notificação adequada a processo externo sem mecanismos improvisados ([09:09] Diego (TRANSCRICAO.md:L64)).
- <a id="adr-002-alt-03"></a>[ADR-002-ALT-03] **Múltiplos workers com particionamento ou lock pessimista:** adiado. Poderia aumentar vazão, mas exige tratar concorrência e ordenação por `order_id` ([09:12] Diego (TRANSCRICAO.md:L80); [09:13] Diego (TRANSCRICAO.md:L84)).

## Consequências

### Positivas

- <a id="adr-002-pos-01"></a>[ADR-002-POS-01] O polling de 2 segundos é compatível, em condições normais, com a expectativa inferior a 10 segundos.
- <a id="adr-002-pos-02"></a>[ADR-002-POS-02] API e worker podem iniciar, parar e falhar de forma independente.
- <a id="adr-002-pos-03"></a>[ADR-002-POS-03] O worker reutiliza runtime, banco e stack já conhecidos pelo time.
- <a id="adr-002-pos-04"></a>[ADR-002-POS-04] **Análise derivada/proposta:** O single-worker simplifica o primeiro desenho; com o gating proposto, retry bloqueia somente o mesmo destino/pedido, não outros endpoints ou pedidos.

### Negativas

- <a id="adr-002-neg-01"></a>[ADR-002-NEG-01] **Análise derivada/proposta:** Um único worker limita throughput e constitui ponto único de processamento.
- <a id="adr-002-neg-02"></a>[ADR-002-NEG-02] **Análise derivada/proposta:** Polling gera consultas mesmo quando não há eventos.
- <a id="adr-002-neg-03"></a>[ADR-002-NEG-03] **Análise derivada/proposta:** Sem o gating proposto, a ordenação por antiguidade não impede overtaking durante retry.
- <a id="adr-002-neg-04"></a>[ADR-002-NEG-04] **Análise derivada/proposta:** Com o gating, há head-of-line blocking dentro do mesmo destino/pedido até entrega, DLQ ou cancelamento; replay da DLQ pode chegar fora de ordem.
- <a id="adr-002-neg-05"></a>[ADR-002-NEG-05] **Análise derivada/proposta:** Janela de candidatos, claim individual e lease/recuperação após crash exigem aprovação antes do worker; detalhes de deploy não bloqueiam o início do código.
