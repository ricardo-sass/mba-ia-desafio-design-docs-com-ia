<a id="adr-003"></a>

# ADR-003 — Retry com backoff e DLQ

## Status

Aceito parcialmente. Backoff limitado, cinco marcos temporais, DLQ e replay estão aceitos; a quantidade total de envios permanece pendente.

## Contexto

Endpoints de clientes podem ficar indisponíveis por minutos ou horas. Descartar o evento na primeira falha perde notificações; tentar indefinidamente mantém eventos presos e consome recursos sem limite. A reunião registrou indisponibilidades planejadas de até duas horas como cenário real ([09:16] Diego (TRANSCRICAO.md:L100)).

Após esgotar as tentativas, operação e suporte ainda precisam de evidência da falha e de um mecanismo controlado de recuperação.

## Decisão

Adotar uma política limitada com os marcos temporais, nesta ordem: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas ([09:17] Diego (TRANSCRICAO.md:L104); [09:17] Larissa (TRANSCRICAO.md:L108)).

A transcrição não fecha a cardinalidade: Diego fornece cinco intervalos após mencionar a primeira falha ([09:17] Diego (TRANSCRICAO.md:L104)), enquanto Larissa registra “5 tentativas” ([09:17] Larissa (TRANSCRICAO.md:L108)) e repete “total 5 tentativas” no resumo ([09:48] Larissa (TRANSCRICAO.md:L282)). Portanto, ainda deve ser decidido se a política contém cinco envios totais ou uma tentativa inicial seguida de cinco retries. Este ADR não escolhe uma das interpretações.

Esgotada a cardinalidade que vier a ser aprovada, persistir payload, motivo e timestamp em uma tabela `webhook_dead_letter` separada da outbox. Disponibilizar replay manual por endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, restrito à role `ADMIN` e com registro de quem executou a ação ([09:18] Diego (TRANSCRICAO.md:L110); [09:18] Diego (TRANSCRICAO.md:L114); [09:36] Sofia (TRANSCRICAO.md:L210)).

A classificação exata de respostas HTTP e falhas de rede como sucesso, retentável ou permanente permanece pendente; este ADR não a define.

## Alternativas Consideradas

- <a id="adr-003-alt-01"></a>[ADR-003-ALT-01] **Três retries:** descartado por cobrir uma janela curta demais para indisponibilidades conhecidas ([09:16] Diego (TRANSCRICAO.md:L100)).
- <a id="adr-003-alt-02"></a>[ADR-003-ALT-02] **Retry indefinido:** descartado porque um endpoint abandonado manteria o evento pendente para sempre ([09:15] Diego (TRANSCRICAO.md:L96)).
- <a id="adr-003-alt-03"></a>[ADR-003-ALT-03] **Marcar falha permanente na própria outbox:** descartado em favor de uma DLQ separada, que mantém a leitura operacional de pendências mais limpa e preserva evidências para diagnóstico ([09:18] Diego (TRANSCRICAO.md:L110)).

## Consequências

### Positivas

- <a id="adr-003-pos-01"></a>[ADR-003-POS-01] Falhas temporárias recebem marcos de recuperação que chegam a 12 horas após a tentativa anterior e cobrem aproximadamente 15 horas no encadeamento citado.
- <a id="adr-003-pos-02"></a>[ADR-003-POS-02] **Análise derivada/proposta:** Falhas permanentes deixam de disputar a consulta principal de pendências.
- <a id="adr-003-pos-03"></a>[ADR-003-POS-03] A DLQ preserva evidências e oferece recuperação manual auditável.
- <a id="adr-003-pos-04"></a>[ADR-003-POS-04] A política limitada impede retry infinito depois que sua cardinalidade for confirmada.

### Negativas

- <a id="adr-003-neg-01"></a>[ADR-003-NEG-01] **Análise derivada/proposta:** A notificação pode chegar muitas horas depois da mudança original.
- <a id="adr-003-neg-02"></a>[ADR-003-NEG-02] **Análise derivada/proposta:** São necessários estados, agendamento e histórico de tentativas consistentes.
- <a id="adr-003-neg-03"></a>[ADR-003-NEG-03] **Análise derivada/proposta:** Replay pode gerar nova duplicata, coerente com a garantia at-least-once.
- <a id="adr-003-neg-04"></a>[ADR-003-NEG-04] **Análise derivada/proposta:** A implementação permanece bloqueada até os revisores resolverem a contradição entre cinco intervalos e cinco tentativas totais.
- <a id="adr-003-neg-05"></a>[ADR-003-NEG-05] Sem a classificação pendente de respostas, o comportamento do worker ainda não está totalmente implementável.
