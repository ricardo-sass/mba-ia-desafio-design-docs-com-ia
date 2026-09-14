# 📝 Da reunião ao documento: design docs gerados por IA

## 🎯 Sobre o desafio

Este repositório transforma a transcrição de uma reunião técnica e o código de um Order Management System em um pacote de design docs para uma futura feature de webhooks outbound de mudança de status. A entrega documenta produto, arquitetura, decisões e implementação em níveis diferentes, sem alterar a aplicação.

O trabalho teve duas restrições centrais: toda afirmação relevante precisava ter origem verificável na [transcrição](./TRANSCRICAO.md) ou no código, e sugestões/itens adiados não poderiam aparecer como decisões fechadas. O [enunciado original](./ENUNCIADO.md) permanece disponível para consulta.

## 🤖 Ferramentas de IA utilizadas

- **Codex (OpenAI):** leitura do repositório, classificação da transcrição, redação e revisão cruzada dos documentos.
- **Skills OpenSpec no Codex:** `openspec-propose` estruturou proposta, design, especificações e tarefas; `openspec-apply-change` conduziu a execução e o acompanhamento de 81 tarefas: 58 na criação e refinamento do pacote documental e 23 na revisão das referências do Tracker, todas marcadas como concluídas nos respectivos arquivos `tasks.md`.
- **OpenSpec CLI:** criou a mudança, forneceu os templates/contextos e validou os artefatos com modo estrito.

Ferramentas auxiliares não generativas: `rg`, comandos shell somente leitura e verificações de links/contagens foram usados para confirmar caminhos, seções, IDs e percentuais do Tracker.

## 🔄 Workflow adotado

1. **Contextualização:** leitura integral do enunciado, transcrição, configuração OpenSpec e arquivos centrais do OMS.
2. **Classificação da evidência:** separação entre requisito, decisão, alternativa descartada, exclusão, risco e questão aberta; definição de IDs estáveis.
3. **Proposta OpenSpec:** criação de `proposal.md`, `design.md`, três specs normativas iniciais (`design-docs-webhooks-pedidos`, `rastreabilidade-documental` e `relato-processo-ia`) e `tasks.md` para uma mudança exclusivamente documental. A revisão posterior do Tracker acrescentou `auditoria-semantica-tracker`, totalizando quatro specs consolidadas em `openspec/specs/`.
4. **ADRs primeiro:** registro isolado das seis decisões principais, formando o esqueleto arquitetural.
5. **RFC:** consolidação concisa da proposta, alternativas, riscos e pontos que ainda precisam de revisão.
6. **FDD:** detalhamento de fluxos, dados, contratos propostos, erros, resiliência, observabilidade, testes e integração com o código existente.
7. **PRD:** consolidação do problema, público, objetivos, escopo e critérios no nível de produto.
8. **Tracker:** cruzamento de todos os IDs com timestamp/falante ou caminho real, seguido de cálculo automático da cobertura.
9. **README e auditoria:** relato do processo efetivamente executado e revisão da checklist completa.

A interação com a IA foi dirigida por restrições explícitas e verificações objetivas. Cada lote de documentos foi produzido, inspecionado por busca/contagem e só então teve suas tarefas marcadas como concluídas no OpenSpec.

## 💬 Prompts customizados

Os prompts abaixo representam as instruções de trabalho adaptadas e aplicadas ao contexto do Codex/OpenSpec durante a produção.

### 🔎 1. Extração de evidências sem promover sugestões a requisitos

```text
Leia TRANSCRICAO.md integralmente e produza uma matriz com: requisito explícito,
decisão fechada, alternativa descartada, item fora de escopo e questão aberta.
Para cada linha, preserve exatamente um timestamp [hh:mm] e o nome do falante.
Não use prática de mercado como evidência e não transforme frases como “tipo”,
“ou assim”, “eu sugiro” ou “problema do futuro” em contrato confirmado.
Confronte os itens técnicos com caminhos reais do código antes de documentá-los.
```

### 🛠️ 2. FDD acionável com fronteira entre fato e proposta

```text
Gere o FDD da feature outbound de webhooks usando apenas TRANSCRICAO.md e o código
como fontes primárias. Detalhe outbox atômica, worker, retry/DLQ, HMAC, at-least-once,
contratos HTTP, erros WEBHOOK_*, observabilidade e testes. Em “Integração com o sistema
existente”, cite no mínimo quatro caminhos que existam e descreva a integração real.
Marque como “PROPOSTA — REQUER APROVAÇÃO” qualquer path, status HTTP, locking,
classificação de resposta ou formato criptográfico que a reunião não tenha fechado.
```

### 🔗 3. Auditoria de rastreabilidade

```text
Extraia todos os IDs explícitos de PRD, RFC, FDD e ADRs e compare-os ao TRACKER.md.
Falhe se houver ID ausente/duplicado, timestamp sem fala correspondente ou caminho de
código inexistente. Calcule cobertura, proporção TRANSCRICAO e total CODIGO; exija
respectivamente >= 80%, >= 70% e >= 5. Não aceite ENUNCIADO.md como evidência de uma
decisão da feature.
```

## 🔧 Iterações e ajustes

Foram realizadas **nove iterações principais** de produção e revisão:

1. **Separação entre decisão e proposta:** o primeiro desenho identificou que a reunião confirma capacidades do CRUD, mas não fecha seus paths nem a posição de `customer_id`. Os contratos do FDD foram então rotulados como propostas para revisão, preservando como verbalizados `GET /webhooks/:id/deliveries` e `POST /admin/webhooks/dead-letter/:id/replay`.
2. **Validação de caminhos:** a redação inicial nomeava um arquivo futuro para o worker. Como esse arquivo ainda não existe e a entrega não implementa código, a referência foi corrigida para “novo entry point análogo a `src/server.ts`”; apenas caminhos existentes ficaram na seção de integração.
3. **Cobertura conservadora do Tracker:** decisões, riscos, entidades, erros e integrações receberam IDs estáveis, mas a revisão reconheceu que IDs não provam sozinhos a cobertura. A auditoria passou a partir de um inventário manual de itens identificáveis e deixou de declarar 100%.
4. **Ambiguidade de retry:** a transcrição contém cinco intervalos, mas também registra “5 tentativas” e “total 5 tentativas”. A primeira versão escolhia provisoriamente uma tentativa inicial mais cinco retries; a revisão removeu essa escolha de PRD, RFC, ADR-003, FDD e Tracker e manteve a cardinalidade como questão bloqueante.
5. **Acionabilidade do FDD:** classificação HTTP, claim/recuperação, assinatura e rotação estavam apenas listados como bloqueios. A revisão acrescentou defaults completos, premissas e fontes, mantendo-os explicitamente sujeitos à aprovação dos revisores e de Segurança.
6. **Ordenação, lease e semânticas operacionais:** a revisão demonstrou que single-worker não impede overtaking durante backoff e que reservar 20 itens com lease de 60 segundos era incompatível com chamadas sequenciais de até 10 segundos. O FDD passou a propor gating por destino/pedido, janela sem reserva seguida de claim individual, cancelamento de pendências ao desativar/remover e replay repetido idempotente. O Tracker também passou a enumerar os itens sem linha própria.
7. **Horizontes, validação, rotação e granularidade:** retenção deixou de ser tratada como bloqueio da primeira fase; foi proposto um mapper opcional para converter issues Zod de URL/filtro em erros `WEBHOOK_*`; rotações sucessivas passaram a retornar conflito durante o grace period; e decisões, 18 alternativas e 48 consequências dos ADRs receberam rastreabilidade individual, com recálculo do Tracker.
8. **Evidência semântica e consistência do worker:** exclusões compostas foram desagregadas; PRD e FDD passaram a permitir implementação incremental; parâmetros de claim ficaram somente no FDD; critérios derivados foram marcados como condicionais; o baseline `HMAC(rawBody)` foi separado da extensão com timestamp; e foram definidos cenário nominal de latência, ordenação por destino/sequência e finalização transacional das tentativas.

9. **Auditoria integral das referências do Tracker:** a mudança `revisar-referencias-tracker` revisou as 240 linhas principais e os 27 itens complementares contra os documentos completos, a transcrição e o código. Corrigiu fontes parciais, distinguiu falas com timestamp/falante repetidos, decompôs afirmações com fontes distintas e preservou os IDs originais como agrupadores navegáveis. HTTPS/HMAC, payload/headers, snapshot, worker, retry/DLQ/replay e os controles propostos foram conferidos separadamente. Também corrigiu premissas sobre redação de secrets, chave-mestra de ambiente, prefixo de rotas e isolamento do banco de testes. O inventário final possui **471 unidades principais e 20 complementares**, com **95,9% de cobertura**, **424 linhas TRANSCRICAO (90,0%)** e **47 CODIGO (10,0%)**. As duas mudanças passaram em `openspec validate --strict`; IDs, links, âncoras, fontes e aritmética foram verificados. Esses checks estruturais complementam o confronto semântico registrado na [auditoria por unidade](./openspec/changes/archive/2026-09-12-revisar-referencias-tracker/auditoria.md). Aplicação, testes, configurações e transcrição foram preservados em relação à cópia inicial do workspace.

Também foi feita uma revisão de fronteiras: detalhes de payload, erros e algoritmos permaneceram no FDD; o RFC foi mantido conciso; o PRD ficou centrado em valor, escopo e aceite; e os ADRs não foram copiados integralmente para outros documentos.

## 🧭 Como navegar a entrega

Ordem sugerida de leitura:

1. 🎯 [PRD — problema, público, escopo e sucesso](./docs/PRD.md)
2. 🏗️ [RFC — proposta arquitetural para revisão](./docs/RFC.md)
3. ⚖️ [ADRs — decisões isoladas](./docs/adrs/)
   - [ADR-001 — Outbox transacional no MySQL](./docs/adrs/ADR-001-outbox-transacional-no-mysql.md)
   - [ADR-002 — Worker separado com polling](./docs/adrs/ADR-002-worker-separado-com-polling.md)
   - [ADR-003 — Retry com backoff e DLQ](./docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
   - [ADR-004 — HMAC-SHA256 e rotação de secret](./docs/adrs/ADR-004-hmac-sha256-e-rotacao-de-secret.md)
   - [ADR-005 — Entrega at-least-once](./docs/adrs/ADR-005-entrega-at-least-once.md)
   - [ADR-006 — Reuso dos padrões do OMS](./docs/adrs/ADR-006-reuso-dos-padroes-do-oms.md)
4. 🛠️ [FDD — especificação detalhada de implementação](./docs/FDD.md)
5. 🔗 [Tracker — origem de requisitos e decisões](./docs/TRACKER.md)
6. 📦 [Mudança OpenSpec — proposta, design, specs e tarefas](./openspec/changes/archive/2026-09-12-criar-pacote-design-docs-webhooks/)
7. 🔎 [Auditoria semântica — reconciliação dos IDs e evidências](./openspec/changes/archive/2026-09-12-revisar-referencias-tracker/auditoria.md)

O código em `src/`, `prisma/` e `tests/` é contexto somente leitura. Nenhuma capacidade de webhook foi implementada neste desafio.
