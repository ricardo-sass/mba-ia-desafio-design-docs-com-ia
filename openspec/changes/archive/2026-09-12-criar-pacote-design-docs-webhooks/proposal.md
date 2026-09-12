## Why

Três clientes B2B precisam deixar de consultar pedidos por polling e esperam receber mudanças de status em menos de 10 segundos (`TRANSCRICAO.md`, [09:00]–[09:03] Marcos/Sofia). Como as decisões técnicas ficaram apenas na reunião, é necessário convertê-las, junto aos padrões reais do OMS, em um pacote de design docs rastreável e acionável antes que o time inicie a implementação (`TRANSCRICAO.md`, [09:50] Larissa; `ENUNCIADO.md`).

## What Changes

- Produzir `docs/PRD.md`, separando problema, público, objetivos mensuráveis, escopo, requisitos, riscos, critérios de aceitação e validação no nível de produto.
- Produzir `docs/RFC.md`, conciso e voltado à revisão, consolidando a arquitetura proposta, alternativas descartadas, questões em aberto, impactos, riscos e links para decisões relacionadas.
- Produzir `docs/FDD.md`, detalhando fluxos, contratos HTTP, erros `WEBHOOK_*`, resiliência, observabilidade, testes e integração com caminhos reais do OMS, em especial a transação de `OrderService.changeStatus` em `src/modules/orders/order.service.ts`.
- Registrar entre cinco e oito decisões isoladas em ADRs no formato MADR, cobrindo transactional outbox no MySQL, worker separado em polling, retry/DLQ, HMAC e rotação de secret, entrega at-least-once e reuso dos padrões existentes (`TRANSCRICAO.md`, [09:06]–[09:30] e [09:48] Larissa).
- Criar `docs/TRACKER.md` para ligar requisitos, decisões, restrições e trade-offs a timestamps da transcrição ou caminhos reais do código, com as metas de cobertura definidas no `ENUNCIADO.md`.
- Substituir o conteúdo de `README.md` por um relato verificável do processo assistido por IA, incluindo ferramentas, workflow, prompts, iterações e navegação da entrega.
- Revisar o pacote contra todos os critérios de aceite e fontes primárias, desagregando afirmações compostas, delimitando o alcance semântico de cada evidência e removendo contradições/duplicações.
- Manter como não objetivos a implementação da feature e qualquer alteração em `src/`, `prisma/`, `tests/`, configurações ou `TRANSCRICAO.md`. Também preservar como fora do escopo da primeira fase inbound webhooks, exactly-once, Redis, múltiplos workers, email de fallback, dashboard e rate limiting de saída (`TRANSCRICAO.md`, [09:02], [09:07], [09:13], [09:25]–[09:26] e [09:37]–[09:40]).
- Tratar lacunas sem decisão silenciosa e por horizonte: permitir implementação incremental, bloquear apenas o comportamento afetado, fechar limites operacionais antes do deploy e manter retenção definitiva/limpeza como evolução. Detalhes de claim, latência, ordenação e atomicidade ficam no FDD; critérios derivados permanecem condicionais à aprovação.

## Capabilities

### New Capabilities

- `design-docs-webhooks-pedidos`: define a cobertura, as fronteiras e a consistência exigidas do PRD, RFC, FDD e conjunto de ADRs para a futura feature de webhooks outbound.
- `rastreabilidade-documental`: define como cada afirmação relevante do pacote será vinculada à transcrição ou ao código e como a cobertura será validada.
- `relato-processo-ia`: define o conteúdo do README que torna explícito e reproduzível o processo de produção, revisão e navegação dos documentos.

### Modified Capabilities

Nenhuma. Não há especificações OpenSpec existentes nem comportamento da aplicação a ser modificado nesta mudança documental.

## Impact

- **Arquivos produzidos:** `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, de cinco a oito arquivos em `docs/adrs/` e `README.md`.
- **Fontes e dependências:** `ENUNCIADO.md`, `TRANSCRICAO.md`, código e testes existentes. A documentação deve refletir, entre outros pontos, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/`, `src/shared/logger/index.ts` e `prisma/schema.prisma`.
- **APIs e sistemas:** nenhum endpoint, schema de banco, processo ou dependência será criado nesta mudança; o pacote apenas especificará a solução futura. O estado atual continua sem webhooks, eventos, outbox, worker ou DLQ.
- **Objetivo verificável:** entregar todos os artefatos obrigatórios, cobrir integralmente a checklist do `ENUNCIADO.md` e alcançar no Tracker ao menos 80% dos itens identificáveis, ao menos 70% de linhas com fonte `TRANSCRICAO` e ao menos cinco referências a caminhos reais do código.
- **Riscos:** interpretar menções descartadas como requisitos, preencher lacunas com decisões inventadas, duplicar conteúdo entre documentos e citar arquivos inexistentes. As mitigações são revisão por timestamp, inspeção do código, separação explícita por nível documental e auditoria final automatizada/manual.
- **Prazo e revisão futura documentados:** a estimativa aprovada para a implementação é de três sprints e inclui ao menos dois dias úteis de revisão de segurança por Sofia (`TRANSCRICAO.md`, [09:45]–[09:49]); isso será registrado, não executado, por esta mudança.
