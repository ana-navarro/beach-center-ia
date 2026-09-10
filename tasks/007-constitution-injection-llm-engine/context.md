# Task 007 — Atualização do Constitution: `beach-center-bff-injection` + `beach-center-bff-llm-engine`

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> **US01** — tarefa de **documentação arquitetural** (não gera código de aplicação nem testes).
> Perguntas bloqueadoras (1–4) **respondidas** — ver "Decisões confirmadas". Restam perguntas
> menores com default proposto (resolver no `/speckit-plan`).

## Título

Oficializar na **Constituição do Projeto** (`.specify/memory/constitution.md`) a nova topologia de
microsserviços: os dois repositórios novos (`beach-center-bff-injection` e
`beach-center-bff-llm-engine`), o desacoplamento da **validação de comprovante de pagamento** do
serviço de pagamentos, o armazenamento de arquivos via **MinIO** (S3-compatible) e o
**processamento assíncrono de IA** (fila + modelo de visão **Moondream**), com comunicação
`Injection → fila → LLM Engine` estritamente por mensageria.

## Contexto de negócio e técnico

- Hoje o **upload/anexo do comprovante** vive em dois lugares: `beach-center-bff-pagamentos`
  (`SubmitManualPaymentUsecase` — reserva comum, move `pending → waiting_approve` e sobe o arquivo
  para o Google Drive ou base64) e `beach-center-bff-agendamentos` (`attach-proof` da
  `ranking_agendamento`). O armazenamento atual usa **Google Drive** (com fallback base64).
- **Problema**: o tempo de inferência de um modelo de visão causaria **timeout/latência** no
  fluxo síncrono do usuário; o Google Drive impõe **rate limit**.
- **Direção nova** (refina a nota antiga em `sistema-pagamento-futuro` — que falava em
  `llama3.2-vision` + Drive): arquitetura **orientada a eventos**. O front **não aguarda a IA**:
  1. `beach-center-bff-injection` recebe o upload (REST), valida estaticamente o arquivo
     (**PDF/JPG/PNG**, **≤ 5 MB**), sobe para o **MinIO** via S3 SDK, grava o registro com status
     **"Pendente / Em Análise"** associando a URL do MinIO, publica um evento na **fila**
     (`{ id_agendamento, url_arquivo }`) e responde **rápido** ("documento em processamento").
  2. `beach-center-bff-llm-engine` é um **worker** que consome a fila, baixa o arquivo do MinIO,
     submete a imagem ao **Moondream** (classificação binária: "é comprovante de transferência /
     PIX?"), e **atualiza a validação**:
     - IA diz **NÃO** → status **"Documento Inválido / Rejeitado"** e **libera a quadra**.
     - IA diz **SIM** → mantém **"Em Análise"** para revisão manual do admin no painel.
- **MinIO** substitui o Google Drive como storage estratégico para contornar o **rate limit** do
  Drive (S3-compatible, self-hosted, sem cota de API externa).
- Comunicação `Injection ↔ LLM Engine` = **exclusivamente por fila** (RabbitMQ **ou**
  BullMQ/Redis), **nunca** REST síncrono.

## Fluxo de comunicação (a ser documentado no Constitution)

```
1. Front-end        --REST (upload multipart)-->   beach-center-bff-injection
2. bff-injection    --S3 SDK (putObject)-------->   MinIO
3. bff-injection    --grava reserva (status: Em Análise + url_arquivo)-->  Database
4. bff-injection    --publish { id_agendamento, url_arquivo }-->  Fila (RabbitMQ | BullMQ/Redis)
5. bff-llm-engine   <--consume-------------------  Fila
6. bff-llm-engine   --S3 SDK (getObject)-------->  MinIO   (download do arquivo)
7. bff-llm-engine   --inferência visual (Moondream: "é comprovante PIX/TED?")
8. bff-llm-engine   --atualiza validação (Rejeitado + libera quadra | mantém Em Análise)-->  Database
```

- Resposta ao front (passo 1) retorna **antes** dos passos 5–8 (assíncrono).
- Passos 4→5: **mensageria**, sem chamada REST entre os dois serviços.

## Definição dos repositórios novos (texto-base para o Constitution)

### `beach-center-bff-injection`
- **Stack**: Node.js, TypeScript. (Arquitetura Hexagonal — Princípio II; ESLint + testes ≥ 80% —
  Princípio III.)
- **Responsabilidade**: porta de entrada para **upload e gestão inicial** de comprovantes de
  pagamento (a parte que hoje está em `pagamentos`/`agendamentos`).
- **Deveres**:
  - Receber a requisição REST do front-end.
  - Validação **estática** do arquivo: só **PDF, JPG, PNG**; tamanho **≤ 5 MB**.
  - Upload seguro para o **MinIO** (S3 SDK).
  - Gravar o registro do agendamento com status **"Pendente / Em Análise"** + URL do MinIO.
  - Publicar evento na **fila** com `{ id_agendamento, url_arquivo }`.
  - Responder sucesso **rápido** ("documento em processamento").

### `beach-center-bff-llm-engine`
- **Stack**: Node.js, TypeScript, **Moondream** (Vision AI, modelo local), Docker.
- **Responsabilidade**: motor de **validação visual assíncrona** de documentos (visão
  computacional).
- **Deveres**:
  - **Worker** consumindo continuamente a fila de validação (RabbitMQ | BullMQ).
  - Baixar o arquivo do MinIO pela URL da mensagem.
  - Submeter a imagem ao **Moondream** com prompt de **classificação binária** ("Este documento é
    um comprovante de transferência bancária ou PIX?").
  - Atualizar o status (ou publicar evento de retorno):
    - **NÃO** → agendamento "Documento Inválido / Rejeitado" + **libera a quadra**.
    - **SIM** → mantém "Em Análise" (aguarda revisão manual do admin).

## Serviço(s) alvo (Princípio I)

| Repositório | Papel nesta task | Justificativa |
|---|---|---|
| **`beach-center-ia`** (root) | **Único alvo de escrita.** Atualizar `.specify/memory/constitution.md`: (a) Princípio I — adicionar os 2 repos à lista de fronteiras do ecossistema; (b) nova seção/princípio descrevendo a **arquitetura assíncrona de validação de comprovante** (REST → MinIO → Fila → IA); (c) Sync Impact Report + bump de versão. | A Constituição vive aqui. É o documento oficial de arquitetura do ecossistema. |
| `services/beach-center-bff-injection` | Só **citado** no Constitution. Diretório já existe (`package.json` bootstrap, sem código). **Nenhum código nesta task.** | Repo novo — implementação é task futura. |
| `services/beach-center-bff-llm-engine` | Só **citado** no Constitution. Idem. | Idem. |
| ~~`beach-center-server`~~ | **Fora de escopo** (C3) — MinIO / RabbitMQ / Redis no `docker-compose.dev` = task de infra separada. | Princípio IV (infra reproduzível). |
| ~~`beach-center-bff-pagamentos`~~ / ~~`beach-center-bff-agendamentos`~~ | **Não tocados nesta task.** A migração de código do fluxo de comprovante é task futura. | — |
| ~~`beach-center-documentation/`~~ | **Não tocado** (sem endpoints ainda). | — |

## Regras de negócio conhecidas

1. Front **não aguarda** a IA — resposta síncrona rápida de `injection`; validação de IA é
   background.
2. Validação estática em `injection`: **PDF / JPG / PNG**, **≤ 5 MB**.
3. Storage = **MinIO** (S3-compatible, self-hosted) — **substitui o Google Drive** por causa do
   **rate limit** do Drive.
4. Fila entre `injection` e `llm-engine` = RabbitMQ **ou** BullMQ/Redis (a escolher — ver
   perguntas). Comunicação entre os dois serviços é **só por fila**, nunca REST síncrono.
5. Payload do evento: `{ id_agendamento, url_arquivo }`.
6. Modelo de IA = **Moondream** (local), prompt de **classificação binária** (comprovante PIX/TED
   sim/não).
7. Resultado da IA:
   - **NÃO** → status "Documento Inválido / Rejeitado" + **libera a quadra**.
   - **SIM** → mantém "Em Análise" (revisão humana do admin).
8. Reembolso continua **sempre** por contato direto usuário↔admin (herança da 006a — nunca
   automático).

## Impacto arquitetural (documental — não há camadas hexagonais nesta task)

Esta é uma task de **edição de documento**. Único arquivo alterado:
`.specify/memory/constitution.md` (repo `beach-center-ia`). Alterações previstas:

- **Sync Impact Report** (comentário no topo): registrar `1.3.1 → 1.4.0` (MINOR), rationale, e
  a adição do Princípio VI + itens no Princípio I.
- **Princípio I (Fronteiras do Ecossistema)** — adicionar à lista de Backend:
  - `services/beach-center-bff-injection`: microsserviço de **ingestão/upload e gestão inicial de
    comprovantes de pagamento** (Node.js/TypeScript).
  - `services/beach-center-bff-llm-engine`: microsserviço **worker de validação visual de
    documentos por IA** (Node.js/TypeScript, Moondream, Docker).
  - Encaixar na regra já existente ("apesar do sufixo `bff`, ... microsserviços puros e
    independentes") — os dois **não** são BFFs.
- **Novo Princípio VI — "Arquitetura Orientada a Eventos para Validação de Comprovantes"**:
  - O fluxo sequencial `Front → REST → injection → MinIO → Fila → llm-engine → MinIO → IA →
    (REST interna) → agendamentos` (os 8 passos da seção "Fluxo de comunicação").
  - **Regra normativa**: a comunicação entre `injection` e `llm-engine` MUST ser por
    **mensageria assíncrona** (RabbitMQ **ou** BullMQ/Redis) — **nunca** REST síncrono.
  - **Regra normativa**: o front **não aguarda** a inferência de IA — `injection` responde
    imediatamente ("em processamento"); a validação é background.
  - **MinIO** (S3-compatible, self-hosted) é o **storage padrão** de comprovantes, **substituindo
    o Google Drive** — justificativa: **rate limit** da API do Drive.
  - `injection` e `llm-engine` **não** escrevem direto na base de reservas: usam a **REST interna
    (`x-api-key`) do `beach-center-bff-agendamentos`** (dono da coleção) — Princípio I preservado.
  - Nota de mapeamento com a máquina de estados da 006a ("Em Análise" ≈ `waiting_approve`;
    "Rejeitado" ≈ `rejected` + libera a quadra).
- **Linha de versão** no rodapé: `Version: 1.4.0 | Ratified: 2026-08-30 | Last Amended:
  <data da emenda>`.
- **Fora desta task**: `beach-center-server` (containers MinIO/fila) e Princípio IV não são
  editados aqui (C3).

## Decisões confirmadas (rodada 1 — `/speckit-task`)

| # | Pergunta | Decisão |
|---|---|---|
| **C1** | Nomenclatura vs. Princípio I (`bff-` = BFF?) | **Manter** `beach-center-bff-injection` / `beach-center-bff-llm-engine` (batem com os diretórios + `package.json` já criados). O Constitution descreve os dois como **microsserviços puros e independentes**, aplicando a regra já existente no Princípio I de que o prefixo `bff-` é legado e o projeto **não** usa o pattern BFF. O texto do enunciado que chama `injection` de "Backend for Frontend" é reescrito como "porta de entrada / serviço de ingestão". |
| **C2** | Estrutura + versão da emenda | **Novo Princípio VI** — *"Arquitetura Orientada a Eventos para Validação de Comprovantes"* — com o fluxo `REST → MinIO → Fila → IA`, a regra de **mensageria-only** entre `injection` e `llm-engine`, e o MinIO como storage padrão. Os 2 repos também entram na lista do **Princípio I**. Bump **MINOR: 1.3.1 → 1.4.0**. Atualizar o Sync Impact Report. |
| **C3** | Escopo de escrita | **Só** `.specify/memory/constitution.md`. Infra (MinIO/RabbitMQ/Redis no `docker-compose.dev`) e bootstrap/README dos 2 repos = **tasks futuras**. |
| **C5** | Fronteira `injection`/`llm-engine` → Database | **REST interna** para o `beach-center-bff-agendamentos` (`x-api-key`): ele continua **dono único** da coleção de reservas. `injection` chama para criar/marcar o agendamento como "em análise" + URL do MinIO; `llm-engine` chama para marcar "rejeitado + libera quadra" ou "mantém em análise". **Só** a comunicação `injection ↔ llm-engine` é proibida de ser REST (obrigatoriamente fila). |

## Perguntas menores em aberto (default proposto — fechar no `/speckit-plan`)

| # | Pergunta | Default proposto |
|---|---|---|
| **Q4** | Fila: RabbitMQ **ou** BullMQ/Redis? | Constitution documenta **as duas como aceitáveis** (mensageria assíncrona); a task de implementação escolhe **uma**. A regra normativa é "comunicação por fila, nunca REST síncrono", não a marca. |
| **Q6** | Mapeamento com a máquina de estados da 006a | Constitution cita, em nota, o mapeamento **"Em Análise" ≈ `waiting_approve`** e **"Documento Inválido/Rejeitado" ≈ `rejected`** (+ liberar quadra = mesma semântica de `rejected`/`cancelled` da 006a). O detalhe fino fica para a task de implementação. |
| **Q7** | `llm-engine`: atualiza o banco **ou** publica evento de retorno? | Consistente com **C5**: `llm-engine` **chama o `agendamentos` via REST interna** para atualizar o status. "Evento de retorno" fica registrado como **evolução futura possível**, não normativa. |
| **Q8** | Como o Moondream roda | Constitution diz **"modelo de visão local (Moondream), servido por container próprio"** — sem fixar Ollama / servidor HTTP / pacote npm. Detalhe = task de implementação do `llm-engine`. |
| **Q9** | Ciclo Speckt para task de doc | Ciclo efetivo: **`task → plan → implement (aplicar as edições no `constitution.md`) → validate → complete (commit/push)`**. `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test` = **N/A** (sem código de aplicação). O `plan.md` traz checklist de edições + Critérios de Aceite verificáveis sobre o **conteúdo** do documento (DoD do enunciado). |
| **Q10** | Tela de status no `beach-center-app` (Aprovar/Recusar) | **Fora de escopo** — frontend + task futura. |
| **Q11** | RAG / pastas "aprovado-rejeitado" como base de treino (nota em `sistema-pagamento-futuro`) | **Fora de escopo** desta US01 — não aparece no enunciado. O Constitution pode citar "revisão manual do admin" e deixar RAG/aprendizado como direção futura, sem normatizar. |

## Notas

- Refina/atualiza a memória `sistema-pagamento-futuro` (que mencionava `llama3.2-vision` + Google
  Drive + RAG). O **RAG** e o fluxo "aprovado/rejeitado como base de treino" **não** aparecem no
  enunciado da US01 — tratar como fora de escopo aqui (task futura), a menos que você diga o
  contrário.
- Os diretórios `services/beach-center-bff-injection` e `services/beach-center-bff-llm-engine` já
  existem (cada um com um `package.json` bootstrap e um commit `first commit`), mas **não** são
  referenciados como submódulos no `beach-center-ia` ainda (não estão no índice do root).
- Sem CI em nenhum repo.

## Próximo passo

Perguntas bloqueadoras (C1, C2, C3, C5) **fechadas**. Rodar **`/speckit-plan`** — ele confirma os
defaults Q4/Q6–Q11 e gera o checklist de edições + Critérios de Aceite verificáveis sobre o
conteúdo do `constitution.md` (cobrindo o DoD do enunciado).
