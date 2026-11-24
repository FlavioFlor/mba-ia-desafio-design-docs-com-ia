# Da Reunião ao Documento — Design Docs Gerados por IA

Repositório com a entrega do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"** (MBA Full Cycle). O ponto de partida: um OMS Node.js/TypeScript funcional em produção, a transcrição literal de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`) sobre uma feature nova (Sistema de Webhooks de Notificação de Pedidos) — e nada escrito além disso.

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o Desafio

A tarefa foi transformar aquela transcrição (e o código existente) em um pacote completo de design docs — PRD, RFC, FDD, 7 ADRs, Tracker de rastreabilidade e este README — em nível acionável para o time de engenharia começar a implementar. O desafio tem duas camadas: produzir a documentação e, ao mesmo tempo, exercitar o papel de **maestro de IA** — dirigir a produção com prompts específicos, revisar criticamente cada saída e iterar até o resultado ficar consistente.

A regra de ouro que guiei o trabalho inteiro: **nenhum requisito, decisão ou restrição sem origem identificável**. Tudo o que está nos documentos aponta para uma fala com timestamp da transcrição ou para um arquivo real do repositório — e o que não tinha origem foi marcado como derivação explícita ou ficou de fora.

## Ferramentas de IA Utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **OpenCode (agente de código, modelo GLM)** | Ferramenta principal: leitura dirigida do repositório (estrutura, módulos, middlewares, schema Prisma), análise da transcrição, extração e filtragem de decisões, redação dos 12 documentos e revisão crítica contra a fonte |
| **Prompts dirigidos (custom)** | Estruturação de cada ciclo: extração com filtragem de descartados, revisão anti-alucinação, normalização de inconsistências (ver seção abaixo) |
| **Script de validação (Node/pwsh, gerado com IA)** | Verificação automática dos critérios de aceite: seções obrigatórias, contagem de requisitos, formato de timestamps, existência dos caminhos de arquivo citados e das 6 decisões principais nos ADRs |

## Workflow Adotado

Ordem de produção (a sugerida pelo desafio, validada na prática):

1. **Contextualização**: leitura integral da transcrição e exploração do código com foco nos pontos que a reunião cita — `order.service.ts` (transação do `changeStatus`), classes de erro, middlewares (`auth`, `error`, `validate`, `request-logger`), schema Prisma, padrão de módulos e testes. Sem essa etapa, qualquer geração posterior seria genérica.
2. **Extração dirigida de decisões**: varredura da transcrição para separar (a) decisões fechadas, (b) requisitos funcionais, (c) restrições, (d) pontos em aberto e (e) **descartados/aditados** — o filtro do que NÃO entra foi tratado como cidadão de primeira classe, não como afterthought.
3. **ADRs primeiro** (7): cada decisão fechada virou um ADR com alternativas reais da reunião e trade-offs explícitos — o esqueleto do resto do pacote.
4. **RFC**: consolidação da proposta técnica em cima dos ADRs, com as alternativas descartadas e as 4 questões em aberto da reunião. Deliberadamente curto (nível de arquitetura), empurrando o detalhe para o FDD.
5. **FDD**: o documento mais fundo — fluxos passo a passo, modelo de dados, 7 contratos HTTP com payloads de exemplo, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção de integração com 15 pontos reais do código.
6. **PRD**: por último entre os grandes — consolidou, em linguagem de produto, o que já estava decidido tecnicamente (evitou assim a duplicação RFC↔PRD).
7. **Tracker**: varredura dos documentos prontos, linha a linha, com verificação de timestamp e de existência de arquivo.
8. **README**: documentação do processo em si (este arquivo).
9. **Revisão final**: script de validação da checklist de critérios de aceite + leitura cruzada (consistência entre documentos, fronteira PRD/RFC/FDD).

## Prompts Customizados

Os três prompts estruturantes do processo (adaptados para este contexto, usados entre ciclos de revisão):

**Prompt 1 — Extração com filtro de descartados** (ciclo 2, antes dos ADRs):

```
Leia TRANSCRICAO.md inteira e produza um inventário em 5 blocos, com timestamp
e falante em cada item:
1. DECISÕES FECHADAS (com a fala que fechou a decisão — procure por "decidido",
   "anotado", "tá fechado", confirmação de mais de uma pessoa);
2. REQUISITOS FUNCIONAIS explícitos (endpoints, campos, comportamentos);
3. RESTRIÇÕES e requisitos não funcionais (números, limites, prazos);
4. PONTOS EM ABERTO (adiados, "observar e decidir depois");
5. DESCARTADOS/ADIADOS explícitos — CRÍTICO: liste TUDO que foi recusado
   na reunião (email, dashboard, rate limiting, arquivamento, multi-worker,
   inbound, etc.); nada disso pode aparecer como requisito nos documentos.
Ignore amenidades da conversa. Não invente requisito que não tenha fala
de origem. Se um número aparecer, anote a fala exata.
```

**Prompt 2 — Revisão anti-alucinação com o tracker como juiz** (ciclos de revisão):

```
Para cada afirmação do documento gerado, pergunte: qual fala da transcrição
(com [hh:mm] Falante) ou qual arquivo do repositório sustenta isso?
Se você não conseguir preencher a coluna "Localização" do TRACKER.md para
um item, esse item é suspeito de alucinação: ou remova, ou reescreva a
partir da fala real, ou marque explicitamente como "(derivado de ...)".
Verifique também o inverso: nenhuma ideia descartada na reunião apareceu
como requisito? Nenhum arquivo citado é inexistente no repositório?
```

**Prompt 3 — Normalização da inconsistência da agenda de retry** (iteração 2, detalhes abaixo):

```
Diego diz "5 tentativas" [09:15] e depois "1m/5m/30m/2h/12h, total de quase
15 horas ENTRE PRIMEIRA FALHA E ÚLTIMA TENTATIVA" [09:17]. 1+5+30+120+720
min = 14,6h. Faça a conta e me diga: quantos envios são necessários para a
frase de Diego ser verdadeira? Depois, verifique se PRD, RFC, FDD e ADR-003
estão consistentes entre si sobre essa agenda — quero a mesma interpretação
em todos, com a tabela de agenda explícita no FDD.
```

## Iterações e Ajustes

Foram **4 ciclos principais** de geração → revisão crítica → ajuste até a versão final:

**Iteração 1 — Geração inicial e o problema da superficialidade.** A primeira passada de extração tratou tudo que foi *mencionado* na reunião como conteúdo de mesma importância. A revisão crítica mostrou mistura entre decisão fechada ("decidido", confirmado por 2+ pessoas) e mera hipótese discutida e rejeitada (trigger de banco, 3 tentativas, retry infinito, secret global). Correção: refinar o prompt de extração (Prompt 1) para exigir a *fala que fechou* cada decisão e para listar os descartados em bloco separado — os descartados alimentaram diretamente as seções "Fora de escopo" do PRD e "Alternativas consideradas" do RFC/ADRs.

**Iteração 2 — Inconsistência matemática na agenda de retry.** Ao cruzar [09:15] ("5 tentativas") com [09:17] ("1m/5m/30m/2h/12h ≈ 15h entre primeira falha e última tentativa"), a conta não fechava com a leitura literal (5 envios no total dariam ~2h36 até a última tentativa). A IA foi instruída a refazer a aritmética (Prompt 3): a única leitura que fecha a frase de Diego é **1 tentativa inicial + 5 retentativas** (a última ocorre ~14,6h após a primeira falha). A agenda explícita foi normalizada no FDD (§5.3) e os demais documentos passaram a usar a mesma formulação. Foi exatamente o tipo de detalhe que, sem revisão crítica, viraria inconsistência entre documentos.

**Iteração 3 — Excesso de detalhes derivados apresentados como decisão da reunião.** Na primeira versão do FDD/tracker, escolhas de modelagem sem fala de origem (nome do endpoint de rotação, nome do campo `statusFilter`, códigos como `WEBHOOK_INACTIVE`, 2xx=sucesso) apareceram como se tivessem sido decididas na call. Correção: aplicar o Prompt 2 sistematicamente — cada item derivado passou a ser marcado "(derivado)" no tracker e no FDD, mantendo a fronteira entre "a reunião decidiu" e "o design propõe seguindo o padrão citado".

**Iteração 4 — Validação automatizada final.** Script de verificação da checklist de critérios de aceite (seções obrigatórias em cada documento, ≥8 FRs, ≥2 alternativas e questões em aberto no RFC, ≥4 endpoints no FDD, 5–8 ADRs, 70%+ de timestamps válidos no tracker, existência dos caminhos de arquivo citados). A rodada apontou ajustes de balanceamento do tracker (mais linhas CODIGO foram adicionadas para cobrir as referências explícitas dos ADRs) e evidenciou a ausência de link explícito entre RFC e ADRs no corpo do texto — corrigidos antes do commit final.

## Como Navegar a Entrega

```
.
├── README.md                     # este arquivo (processo)
├── TRANSCRICAO.md                # fonte primária (não alterada)
├── docs/
│   ├── PRD.md                    # produto: problema, escopo, requisitos, métricas
│   ├── RFC.md                    # arquitetura: proposta, alternativas, questões em aberto
│   ├── FDD.md                    # implementação: fluxos, contratos, erros, integração
│   ├── TRACKER.md                # rastreabilidade item ⇄ fala/código
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-com-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-do-projeto.md
│       └── ADR-007-snapshot-de-payload-na-insercao.md
├── src/                           # código base (não alterado)
├── prisma/                        # código base (não alterado)
└── tests/                         # código base (não alterado)
```

**Ordem sugerida de leitura:**

1. `docs/PRD.md` — o **porquê** (problema dos 3 clientes B2B, escopo e o que ficou de fora).
2. `docs/RFC.md` — o **o que propomos** (visão de arquitetura + alternativas descartadas + questões em aberto).
3. `docs/adrs/` — o **por que de cada decisão** (na ordem 001→007; 001–005 cobrem as decisões principais, 006–007 os desdobramentos).
4. `docs/FDD.md` — o **como construir** (fluxos, contratos HTTP, matriz de erros, integração com o código).
5. `docs/TRACKER.md` — a **prova de origem** de cada item (leitura de auditoria; útil ao dúvidar de qualquer afirmação dos outros documentos).
6. Este README — o processo por trás do pacote.
