# ADR-004 — HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h" [09:22] Sofia; confirmado no resumo final [09:48])

## Contexto

Os eventos carregam dados de pedidos e serão entregues em endpoints fora da infraestrutura da empresa. O cliente precisa conseguir validar que a requisição veio realmente de nós e que ninguém adulterou o payload no meio do caminho ([09:19] Sofia). Além da autenticação, há histórico de incidente: um cliente já vazou secret em log de aplicação dele ([09:22] Diego) — o que torna o risco de comprometimento de credencial concreto, não teórico.

Sofia levantou ainda que uma secret global da plataforma multiplicaria o impacto de qualquer vazamento ("se vaza uma, vaza tudo" — [09:21] Sofia).

## Decisão

**HMAC-SHA256 sobre o corpo do request**, com a assinatura enviada no header `X-Signature`; o cliente verifica do lado dele ([09:20] Sofia, [09:20] Diego — "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca").

Complementos fechados na sequência:

- **Secret única por endpoint de webhook** (não global): a tabela de configuração armazena url + secret + customer_id + estado ativo ([09:21] Sofia/Bruno).
- **Rotação de secret**: endpoint para o cliente pedir uma nova secret pela API. Ao rotacionar, a antiga permanece válida por **24 horas em paralelo** (grace period para migrar os sistemas dele); depois, a antiga morre ([09:21] Sofia).
- **TLS obrigatório**: a URL do webhook deve ser `https`; cadastro com `http` é recusado com erro de validação — tratado como validação no schema Zod, não decisão arquitetural ([09:23] Sofia).
- **Limite de payload de 64KB com erro** (não truncamento): se chegou a esse tamanho, tem algo errado ([09:23]–[09:24] Sofia/Diego). Requisito não funcional complementar.

## Alternativas Consideradas

1. **Secret global compartilhada por toda a plataforma** — descartada: viola o princípio de isolamento levantado por Sofia; o vazamento de uma única credencial comprometeria todas as integrações ([09:21] Sofia). Trade-off: gestão operacional simples (uma secret) vs. raio de explosão total em caso de vazamento — e vazamento já ocorreu na prática ([09:22] Diego).
2. **Não assinar (confiar apenas em TLS + URL secreta)** — descartada implicitamente pela análise de Sofia: TLS protege o transporte, mas não prova a origem nem detecta adulteração por quem conhece a URL; a assinatura HMAC cobre origem e integridade do payload ([09:19]–[09:20] Sofia). Trade-off: zero gestão de credenciais vs. cliente incapaz de verificar autenticidade.

## Consequências

**Positivas:**

- Padrão de mercado: qualquer cliente sério tem biblioteca HMAC-SHA256 disponível ([09:20] Sofia).
- Comprometimento de uma secret afeta um único endpoint de um único cliente — isolamento do risco ([09:21] Sofia).
- Rotação com grace period de 24h permite reagir a vazamentos sem quebrar a integração do cliente ([09:21] Sofia).
- Headers de assinatura permitem ao cliente detectar replay attack (via `X-Timestamp` — [09:44] Diego).

**Negativas:**

- Gestão de N secrets (uma por endpoint) em vez de uma: modelagem e operação mais complexas.
- A janela de 24h com duas secrets válidas é um compromisso de segurança consciente: durante o grace period, a secret antiga (potencialmente vazada) ainda assina ([09:21] Sofia).
- Responsabilidade de verificação recai no cliente; verificação mal implementada do lado dele não é detectável por nós.

## Referências

- Transcrição: [09:19]–[09:24] (Sofia, Bruno, Diego, Larissa).
- Código: `src/modules/orders/order.schemas.ts` (padrão de validação Zod a estender para URL https), `src/shared/errors/http-errors.ts` (`ValidationError` para recusa de URL http).

## Relacionados

- ADR-005 (headers do envio, incluindo X-Signature)
- ADR-006 (validações no padrão Zod do projeto)
