<a id="adr-004"></a>

# ADR-004 — HMAC-SHA256 e rotação de secret

## Status

Aceito, com detalhes de contrato pendentes de revisão de segurança.

## Contexto

Os webhooks transportam dados de pedidos para sistemas fora da infraestrutura do OMS. O consumidor precisa verificar a origem e detectar alteração do corpo. Uma credencial global ampliaria o impacto do vazamento de um único cliente ([09:19] Sofia (TRANSCRICAO.md:L118); [09:21] Sofia (TRANSCRICAO.md:L126)).

Clientes também precisam rotacionar credenciais sem interromper entregas durante a atualização de seus sistemas.

## Decisão

Assinar o corpo efetivamente enviado com HMAC-SHA256 e enviar a assinatura em `X-Signature`. Cada endpoint terá uma secret única, gerada pelo OMS e retornada na criação da configuração ([09:22] Sofia (TRANSCRICAO.md:L134); [09:20] Sofia (TRANSCRICAO.md:L120); [09:31] Marcos (TRANSCRICAO.md:L184)).

Na rotação, uma nova secret passa a ser emitida e a anterior permanece válida em paralelo por 24 horas; depois desse período, a anterior deixa de ser aceita ([09:21] Sofia (TRANSCRICAO.md:L130)). URLs de destino devem usar HTTPS, e cadastros com `http` devem ser rejeitados ([09:23] Sofia (TRANSCRICAO.md:L138)).

O baseline decidido assina somente o corpo. Incluir `X-Timestamp` na mensagem do HMAC seria alteração de contrato, apresentada separadamente em FDD-PROP-03 e dependente de aprovação/versionamento com consumidores. A codificação exata de `X-Signature`, armazenamento/criptografia das secrets, comportamento de exibição após a criação e semântica anti-replay deverão ser fechados com Sofia antes da implementação. Como default sujeito a aprovação, FDD-PROP-04 rejeita uma segunda rotação enquanto a secret anterior ainda estiver nas 24 horas de validade. A revisão de segurança deve reservar ao menos dois dias úteis antes do deploy ([09:46] Sofia (TRANSCRICAO.md:L274)).

## Alternativas Consideradas

- <a id="adr-004-alt-01"></a>[ADR-004-ALT-01] **Secret global da plataforma:** descartada porque um vazamento comprometeria todos os endpoints ([09:21] Sofia (TRANSCRICAO.md:L126)).
- <a id="adr-004-alt-02"></a>[ADR-004-ALT-02] **Análise derivada/proposta:** **Ausência de assinatura:** descartada porque o cliente não conseguiria autenticar a origem nem a integridade do payload ([09:19] Sofia (TRANSCRICAO.md:L118); [09:20] Sofia (TRANSCRICAO.md:L120)).
- <a id="adr-004-alt-03"></a>[ADR-004-ALT-03] **Análise derivada/proposta:** **Invalidar a secret antiga imediatamente:** não escolhida; reduziria a janela de exposição, mas impediria migração coordenada sem indisponibilidade. O grace period aceito é de 24 horas.

## Consequências

### Positivas

- <a id="adr-004-pos-01"></a>[ADR-004-POS-01] **Consequência derivada:** Consumidores podem verificar autenticidade e integridade com bibliotecas amplamente disponíveis.
- <a id="adr-004-pos-02"></a>[ADR-004-POS-02] O isolamento por endpoint reduz o raio de impacto de vazamentos.
- <a id="adr-004-pos-03"></a>[ADR-004-POS-03] A rotação com coexistência permite migração sem corte abrupto.
- <a id="adr-004-pos-04"></a>[ADR-004-POS-04] **Consequência derivada:** HTTPS protege o transporte.

### Negativas

- <a id="adr-004-neg-01"></a>[ADR-004-NEG-01] **Análise derivada/proposta:** O OMS passa a custodiar material secreto sensível e precisa impedir exposição em logs e respostas indevidas.
- <a id="adr-004-neg-02"></a>[ADR-004-NEG-02] Durante 24 horas, duas secrets permanecem válidas.
- <a id="adr-004-neg-03"></a>[ADR-004-NEG-03] **Análise derivada/proposta:** Qualquer diferença na serialização do corpo pode invalidar a assinatura.
- <a id="adr-004-neg-04"></a>[ADR-004-NEG-04] **Análise derivada/proposta:** Os detalhes pendentes, inclusive rotação sucessiva, impedem considerar o contrato criptográfico completo até a revisão de segurança.
