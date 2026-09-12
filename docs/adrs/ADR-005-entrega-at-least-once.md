<a id="adr-005"></a>

# ADR-005 — Entrega at-least-once

## Status

Aceito.

## Contexto

Como cenário derivado da possibilidade confirmada de duplicatas, uma falha após o cliente receber a requisição, mas antes de o worker registrar sucesso, torna impossível saber com certeza se a entrega ocorreu. Repetir a tentativa preserva a notificação, porém pode entregar o mesmo evento mais de uma vez.

Garantir exactly-once exigiria coordenação entre OMS e consumidor, aumentando significativamente a complexidade ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L154)).

## Decisão

Oferecer garantia at-least-once (não promete entrega eventual a endpoint permanentemente indisponível; retry é limitado e DLQ preserva a falha): eventos confirmados pelo OMS não devem ser silenciosamente perdidos, mas podem ser entregues mais de uma vez. Gerar um UUID imutável quando o evento entra na outbox e enviar esse identificador no corpo como `event_id` e no header `X-Event-Id` em todas as tentativas do mesmo evento ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:25] Diego (TRANSCRICAO.md:L154); [09:43] Diego (TRANSCRICAO.md:L256); [09:44] Diego (TRANSCRICAO.md:L262)).

O consumidor será responsável por deduplicar usando `X-Event-Id`, e essa responsabilidade deverá estar destacada no contrato público ([09:25] Diego (TRANSCRICAO.md:L150); [09:26] Marcos (TRANSCRICAO.md:L156)).

## Alternativas Consideradas

- <a id="adr-005-alt-01"></a>[ADR-005-ALT-01] **Exactly-once:** descartado por depender de coordenação distribuída com sistemas de clientes e não justificar a complexidade nesta fase ([09:25] Diego (TRANSCRICAO.md:L154)).
- <a id="adr-005-alt-02"></a>[ADR-005-ALT-02] **Análise derivada/proposta:** **At-most-once:** descartado porque falhas ambíguas poderiam causar perda definitiva da notificação.
- <a id="adr-005-alt-03"></a>[ADR-005-ALT-03] **Análise derivada/proposta:** **Novo ID a cada retry:** descartado por impedir que o consumidor reconheça tentativas duplicadas do mesmo evento.

## Consequências

### Positivas

- <a id="adr-005-pos-01"></a>[ADR-005-POS-01] **Análise derivada/proposta:** Falhas ambíguas podem ser retentadas sem desistir da notificação.
- <a id="adr-005-pos-02"></a>[ADR-005-POS-02] Um UUID estável oferece uma chave simples de deduplicação ao consumidor.
- <a id="adr-005-pos-03"></a>[ADR-005-POS-03] **Análise derivada/proposta:** O modelo é compatível com retry, DLQ e replay.

### Negativas

- <a id="adr-005-neg-01"></a>[ADR-005-NEG-01] Consumidores precisam implementar idempotência; quem não o fizer pode processar o mesmo evento mais de uma vez.
- <a id="adr-005-neg-02"></a>[ADR-005-NEG-02] O OMS não promete ausência de duplicatas.
- <a id="adr-005-neg-03"></a>[ADR-005-NEG-03] **Análise derivada/proposta:** Replay manual e recuperação após crash podem aumentar a ocorrência de duplicatas.
- <a id="adr-005-neg-04"></a>[ADR-005-NEG-04] A documentação externa precisa explicar claramente a semântica e a retenção recomendada da chave, sem transformá-la em garantia de exactly-once. A retenção da chave é recomendação ainda sem prazo definido.
