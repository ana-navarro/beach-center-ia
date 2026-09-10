# Plan — Task 007: Atualização do Constitution (`beach-center-bff-injection` + `beach-center-bff-llm-engine`)

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`.
> **Task doc-only** — edita **um único arquivo**: `.specify/memory/constitution.md`.
> Perguntas bloqueadoras (C1, C2, C3, C5) fechadas no `/speckit-task`. Defaults Q4/Q6–Q11
> confirmados abaixo (seção "Decisões de planejamento").

---

## Contexto Técnico

### Serviço(s) alvo e justificativa (Princípio I)

| Repositório | Alteração | Justificativa da fronteira |
|---|---|---|
| **`beach-center-ia`** (root) — `.specify/memory/constitution.md` | **Único arquivo editado.** Amência formal da Constituição: novo **Princípio VI**, itens no Princípio I, Sync Impact Report, linha de versão. | A Constituição do ecossistema vive aqui. É o documento normativo de arquitetura. Nenhum outro repositório é dono deste texto. |
| `services/beach-center-bff-injection` | **Somente citado** no texto. Diretório já existe (`package.json` bootstrap, commit `first commit`, sem código, não é submódulo do root ainda). | Repo novo — bootstrap/implementação é **task futura**. |
| `services/beach-center-bff-llm-engine` | **Somente citado** no texto. Idem. | Idem. |
| ~~`beach-center-server`~~ | **Fora de escopo (C3).** Containers MinIO / RabbitMQ / Redis no `docker-compose.dev` = task de infra separada. | Princípio IV — não editado aqui. |
| ~~`beach-center-bff-pagamentos`~~ / ~~`beach-center-bff-agendamentos`~~ / ~~`beach-center-app`~~ / ~~`beach-center-documentation/`~~ | **Não tocados.** Migração de código do fluxo de comprovante e telas de admin = tasks futuras. | — |

### Stack e bibliotecas envolvidas

- **Nenhuma dependência de código.** A entrega é texto Markdown em `.specify/memory/constitution.md`.
- Tecnologias **citadas** no novo texto (não instaladas nesta task):
  - `beach-center-bff-injection`: **Node.js, TypeScript** (+ Arquitetura Hexagonal — Princípio II;
    ESLint + testes ≥ 80% — Princípio III). Cliente **S3 SDK** para o MinIO. Cliente de fila
    (RabbitMQ **ou** BullMQ/Redis).
  - `beach-center-bff-llm-engine`: **Node.js, TypeScript, Moondream** (modelo de visão local,
    servido por container), **Docker**. Consumidor de fila. Cliente **S3 SDK** para o MinIO.
  - **MinIO** — object storage S3-compatible, self-hosted (substitui o Google Drive).
  - **Mensageria** — RabbitMQ ou BullMQ sobre Redis.

### Decisões de planejamento (confirmam os defaults Q4/Q6–Q11 do `context.md`)

| # | Decisão para o texto da Constituição |
|---|---|
| **Q4** | O Princípio VI cita **RabbitMQ ou BullMQ/Redis** como opções aceitáveis de mensageria assíncrona. A regra normativa (`MUST`) é *"comunicação `injection ↔ llm-engine` por fila, nunca REST síncrono"* — a marca da fila fica a critério da task de implementação. |
| **Q6** | Nota (não-normativa) de mapeamento com a máquina de estados da task 006a: **"Em Análise" ≈ `waiting_approve`**; **"Documento Inválido / Rejeitado" ≈ `rejected`** (+ liberar a quadra, mesma semântica de `rejected`/`cancelled`). O detalhe fino é da task de implementação. |
| **Q7** | O `llm-engine` **atualiza o resultado chamando o `beach-center-bff-agendamentos` via REST interna** (coerente com C5). "Publicar evento de retorno" fica registrado como **evolução futura possível**, não `MUST`. |
| **Q8** | Texto: *"modelo de visão local (**Moondream**), servido por container próprio"* — sem fixar Ollama / servidor HTTP / pacote npm. |
| **Q9** | Ciclo Speckt **reduzido** para esta task: `task → plan → implement (aplicar as edições no `.md`) → validate → complete (commit/push)`. `/speckit-unit-tests`, `/speckit-component-tests`, `/speckit-test` = **N/A** (sem código de aplicação). |
| **Q10** | Telas do `beach-center-app` (listagem de reservas, botões Aprovar/Recusar) = **fora de escopo** — frontend + task futura. Menção máxima no texto: *"aguarda a revisão manual do admin no painel"*. |
| **Q11** | **RAG / aprendizado com comprovantes aprovados-rejeitados** = **fora de escopo** desta US01. O Princípio VI pode citar *"revisão manual do admin, com migração gradual para validação automática"* como direção, sem normatizar RAG. |

---

## Constitution Check

> Task doc-only. A "conformidade" aqui é dupla: (a) o **processo** de emenda segue a Governança;
> (b) o **texto novo** não contradiz nenhum princípio existente.

| Princípio | Resultado | Justificativa |
|---|---|---|
| **I — Fronteiras do Ecossistema** | **CONFORME** | A task **amplia** a lista de fronteiras (2 microsserviços novos) e **reforça** a regra existente: apesar do prefixo `bff-`, `injection` e `llm-engine` são **microsserviços puros e independentes** (C1). A regra C5 (`injection`/`llm-engine` escrevem no domínio de reservas **apenas** via REST interna do `beach-center-bff-agendamentos`) **preserva** o `agendamentos` como dono único da coleção. A regra de mensageria-only entre os dois novos serviços **fortalece** o isolamento. |
| **II — Arquitetura Hexagonal** | **CONFORME (não impactado)** | Nenhum código nesta task. O Princípio VI é **complementar** ao II: descreve a **topologia entre serviços**, não a estrutura interna. O texto deixa explícito que `injection` e `llm-engine`, sendo Node.js/TypeScript, **continuam sujeitos ao Princípio II** (Ports & Adapters) quando forem implementados. |
| **III — Test-First e Qualidade** | **CONFORME (não impactado)** | Nenhum código/teste gerado. O texto novo reafirma que os 2 repos, quando implementados, seguem o Princípio III (ESLint + cobertura ≥ 80%). O `beach-center-bff-llm-engine` **não** ganha isenção de testes (diferente do `beach-center-whatsapp`, que é JS e isento por decisão anterior) — é TypeScript e entra na regra normal. |
| **IV — Infra Reproduzível** | **CONFORME (não impactado)** | `beach-center-server` não é editado (C3). O Princípio VI **menciona** MinIO/fila como dependências de infra, mas registra explicitamente que a adição ao `docker-compose.dev` (com hot-reload, conforme IV) é **task futura de infra**. |
| **V — Ciclo Speckt** | **CONFORME** | A task segue o ciclo (task → plan → …), com a adaptação Q9 registrada (passos de teste = N/A por ser doc-only). |
| **Governança** | **CONFORME** | A emenda será registrada no próprio arquivo: **Sync Impact Report** atualizado + **linha de versão** `1.3.1 → 1.4.0` (**MINOR** — adição de novo princípio, conforme regra "MINOR: adição de novo princípio"). |

**Resultado: SEM VIOLAÇÕES.** Prosseguir para `/speckit-implement`.

---

## Mapa de Edições no Documento

> Não há "Mapa Arquitetural (Hexagonal)" — não há código. Abaixo, o mapa das edições em
> `.specify/memory/constitution.md`, **em ordem de aplicação**.

### E1 — Sync Impact Report (bloco de comentário no topo, linhas ~1–16)

- `Version change: 1.3.1 → 1.4.0`
- `Rationale (1.3.1 → 1.4.0, MINOR)`: adição do **Princípio VI — Arquitetura Orientada a Eventos
  para Validação de Comprovantes**; inclusão dos microsserviços `beach-center-bff-injection` e
  `beach-center-bff-llm-engine` no Princípio I.
- `Modified principles`: **I** (lista de fronteiras — 2 serviços adicionados).
- `Added sections`: **Princípio VI**.
- `Modified sections`: none (além do Princípio I).
- `Removed sections`: none.
- Mover o conteúdo atual de "Prior amendment history" mantendo o histórico e adicionando a
  entrada `1.3.0 → 1.3.1` no topo do histórico anterior (a linha `1.3.0 → 1.3.1` que hoje está
  na descrição principal desce para o histórico).

### E2 — Princípio I: lista de Backend (linhas ~30–35)

Adicionar dois bullets ao bloco **Backend (BE - services)**, após `beach-center-whatsapp`:

- `services/beach-center-bff-injection`: Microsserviço de **ingestão e gestão inicial de
  comprovantes de pagamento** (upload, validação estática, envio ao storage, publicação na fila).
  Stack: Node.js/TypeScript.
- `services/beach-center-bff-llm-engine`: Microsserviço **worker de validação visual de
  documentos por IA** (visão computacional), operando de forma **assíncrona** via fila. Stack:
  Node.js/TypeScript, Moondream, Docker.

### E3 — Princípio I: bloco ATENÇÃO (linha ~39)

Estender a frase existente ("Apesar do sufixo `bff` ... o projeto **NÃO** utiliza o pattern de
Backend For Frontend (BFF)") para deixar claro que **`beach-center-bff-injection` e
`beach-center-bff-llm-engine` também se enquadram nessa regra** — nomes com `bff-` por
consistência com o legado, mas são **microsserviços puros**. O `injection` **não** é um
"Backend for Frontend": é o **serviço de ingestão** (porta de entrada de upload).

### E4 — Novo Princípio VI (inserir após o Princípio V, antes do `---` da linha ~95)

Título: **`### VI. Arquitetura Orientada a Eventos para Validação de Comprovantes`**

Conteúdo normativo (redação final no `/speckit-implement` — esqueleto abaixo):

1. **Isolamento do fluxo de IA.** A validação de comprovantes de pagamento MUST ser executada
   **fora** do caminho síncrono do usuário. O front-end **MUST NOT** aguardar a inferência de IA.
2. **Serviço de ingestão (`beach-center-bff-injection`).** Deveres:
   - Receber o upload via **REST** (multipart) do front-end.
   - **Validação estática** do arquivo: aceitar **somente** `PDF`, `JPG`, `PNG`; tamanho
     **≤ 5 MB**.
   - Fazer o upload para o **MinIO** (via S3 SDK).
   - Registrar o agendamento como **"Em Análise"** associando a **URL do MinIO** — **via REST
     interna (`x-api-key`) do `beach-center-bff-agendamentos`** (dono da coleção de reservas).
   - **Publicar** um evento na fila com `{ id_agendamento, url_arquivo }`.
   - Responder ao cliente **imediatamente** ("documento em processamento").
3. **Motor de IA (`beach-center-bff-llm-engine`).** Deveres:
   - Atuar como **worker** consumindo continuamente a fila de validação.
   - Baixar o arquivo do **MinIO** pela URL recebida.
   - Submeter a imagem ao **Moondream** (modelo de visão local, servido por container) com um
     **prompt de classificação binária** ("Este documento é um comprovante de transferência
     bancária ou PIX?").
   - Aplicar o resultado **via REST interna do `beach-center-bff-agendamentos`**:
     - **NÃO** → agendamento para **"Documento Inválido / Rejeitado"** e **liberar a quadra**.
     - **SIM** → **manter "Em Análise"** para a revisão manual do admin no painel.
4. **Comunicação entre serviços.**
   - `injection` ↔ `llm-engine`: **MUST** ser por **mensageria assíncrona** (**RabbitMQ** ou
     **BullMQ/Redis**). **MUST NOT** haver chamada **REST síncrona** entre os dois.
   - `injection` → `agendamentos` e `llm-engine` → `agendamentos`: **REST interna** com
     `x-api-key` (nenhum dos dois escreve direto na base de reservas — Princípio I).
5. **Storage — MinIO.** O **MinIO** (S3-compatible, self-hosted) é o **storage padrão** de
   comprovantes, **substituindo o Google Drive**. Justificativa: o Google Drive impõe
   **rate limit** de API incompatível com o volume de upload/download; o MinIO é local e sem
   cota externa.
6. **Fluxo sequencial de referência** (bloco de texto/ASCII):
   ```
   1. Front-end       --REST (upload)-->            beach-center-bff-injection
   2. bff-injection   --S3 SDK (putObject)-->       MinIO
   3. bff-injection   --REST interna (x-api-key)--> beach-center-bff-agendamentos  (status: Em Análise + url_arquivo)
   4. bff-injection   --publish { id_agendamento, url_arquivo }--> Fila (RabbitMQ | BullMQ/Redis)
   5. bff-llm-engine  <--consume---------------      Fila
   6. bff-llm-engine  --S3 SDK (getObject)-->        MinIO  (download)
   7. bff-llm-engine  --inferência visual (Moondream)
   8. bff-llm-engine  --REST interna (x-api-key)-->  beach-center-bff-agendamentos  (Rejeitado + libera quadra | mantém Em Análise)
   ```
7. **Conformidade com os demais princípios.** `injection` e `llm-engine`, sendo Node.js/TypeScript,
   MUST seguir o **Princípio II** (Arquitetura Hexagonal) e o **Princípio III** (ESLint + testes
   unitários com cobertura ≥ 80%). O consumo da fila entra pela camada `applications/controllers/`
   (controllers de evento), como já previsto no Princípio II ("Recebem as requisições
   HTTP/Eventos").
8. **Nota (não-normativa).** Mapeamento com a máquina de estados de pagamento (task 006a):
   "Em Análise" ≈ `waiting_approve`; "Documento Inválido / Rejeitado" ≈ `rejected` (+ libera a
   quadra). A infraestrutura (containers MinIO + fila no `beach-center-server`, Princípio IV) e o
   bootstrap dos dois repositórios são **tasks futuras**. A revisão de aprovação migra
   gradualmente de manual para automática (direção futura — sem normatização de RAG aqui).

### E5 — Linha de versão (rodapé, linha ~166)

`**Version**: 1.4.0 | **Ratified**: 2026-08-30 | **Last Amended**: 2026-09-10`

### E6 — (opcional) Princípio V / Comandos de Desenvolvimento

Se o `/speckit-implement` julgar necessário para coerência: uma frase curta em `/speckit-task`
ou numa nota do Princípio V registrando que **tasks de documentação/arquitetura** rodam o ciclo
reduzido (`task → plan → implement → validate → complete`, sem os passos de teste automatizado).
**Default: NÃO fazer** — manter o escopo mínimo (só o necessário para o DoD). Decidir no
implement/validate.

---

## Checklist de Implementação

> Ordem de dependência. Todos os itens são edições no **mesmo arquivo**
> `.specify/memory/constitution.md`. Nenhum arquivo de código.

- [x] **1.** E2 — Adicionar os bullets de `beach-center-bff-injection` e `beach-center-bff-llm-engine`
  à lista **Backend (BE - services)** do Princípio I (`.specify/memory/constitution.md`, bloco
  "Backend (BE - services)").
- [x] **2.** E3 — Estender o bloco **ATENÇÃO (Regra de Arquitetura)** do Princípio I para cobrir
  explicitamente os dois novos repos (`bff-` legado, microsserviços puros, `injection` ≠ "Backend
  for Frontend").
- [x] **3.** E4 — Inserir o **Princípio VI — Arquitetura Orientada a Eventos para Validação de
  Comprovantes** logo após o Princípio V (antes do `---` que separa "Core Principles" de
  "Comandos de Desenvolvimento"), com os 8 itens do esqueleto (deveres de `injection`, deveres de
  `llm-engine`, regra de mensageria-only, regra de REST interna p/ `agendamentos`, MinIO como
  storage padrão + justificativa do rate limit, bloco do fluxo sequencial, conformidade com II/III,
  nota de mapeamento com a 006a).
- [x] **4.** E1 — Atualizar o **Sync Impact Report** (comentário no topo): `Version change: 1.3.1
  → 1.4.0`; rationale MINOR (novo Princípio VI + 2 serviços no Princípio I); `Modified principles:
  I`; `Added sections: Princípio VI`; empurrar a linha `1.3.0 → 1.3.1` para o "Prior amendment
  history".
- [x] **5.** E5 — Atualizar a **linha de versão** no rodapé para
  `**Version**: 1.4.0 | **Ratified**: 2026-08-30 | **Last Amended**: 2026-09-10`.
- [x] **6.** Revisão de consistência: conferir que nenhuma outra parte do documento referencia
  "5 princípios" ou "Princípios I–V" de forma que fique desatualizada; conferir que o texto novo
  não usa "BFF" no sentido do pattern; conferir a numeração e os cross-references.
- [x] **7.** Verificar que **somente** `.specify/memory/constitution.md` foi alterado
  (`git status --porcelain` deve listar só esse arquivo, fora os diretórios untracked de
  `injection`/`llm-engine` que já existiam).

---

## Validação (`/speckit-validate` — 2026-09-10, modo autônomo "sem aprovação")

- **1 arquivo revisado:** `.specify/memory/constitution.md` — **aprovado sem correções**.
- Conferência: aderência ao plano (E1–E5), 12 ACs (AC-1..AC-12), Princípios I–VI, DoD da US01.
  - Markdown válido: 1 bloco de código fechado (2 fences), headings `I`→`VI` ordenados, `---`
    separando os Core Principles de "Comandos de Desenvolvimento", tabela do rodapé intacta.
  - Sem referências defasadas a "cinco princípios" / "Princípios I–V" no documento.
  - Fronteira preservada (AC-7): `injection`/`llm-engine` escrevem via REST interna do
    `agendamentos`; sem banco compartilhado.
  - `git status`: só `M .specify/memory/constitution.md` (AC-10). Untracked pré-existentes
    (`services/beach-center-bff-injection/`, `.../llm-engine/`, `tasks/007-.../`) intactos.
- **Observação (não-bloqueante):** o Princípio V fala em ciclo estrito para "novas
  funcionalidades". A task 007 é emenda de documentação/governança (não é "nova funcionalidade"),
  então rodar o ciclo reduzido (sem `/speckit-unit-tests` `/speckit-component-tests`
  `/speckit-test`) **não** é violação. **E6 permanece deferido** conforme o default do plano.
- **ESLint: N/A** (Markdown). Alteração **staged** (`git add .specify/memory/constitution.md`),
  **sem commit**.

---

## Desvios / notas de implementação (`/speckit-implement` — 2026-09-10)

- Todos os 7 itens aplicados em `.specify/memory/constitution.md` (1 arquivo, +63 −9 linhas).
- **E6 (opcional) — NÃO feito** (default do plano): não foi adicionada nota no Princípio V sobre
  "ciclo reduzido para tasks de documentação". Escopo mínimo mantido. Pode ser reavaliado no
  `/speckit-validate`.
- **Princípio VI** ficou com **7 blocos numerados** (ingestão / motor de IA / regras de
  comunicação / MinIO / fluxo sequencial / conformidade II–III / notas), não "8 itens" — o
  esqueleto do plano listava 8 tópicos, consolidados em 7 blocos + o bloco de fluxo com 8 passos.
  Conteúdo íntegro.
- O bloco ATENÇÃO do Princípio I foi **estendido na mesma frase** (não viramos um parágrafo
  separado) para citar `injection` / `llm-engine` e negar o rótulo "Backend for Frontend".
- Sync Impact Report: a linha `1.3.0 → 1.3.1 (PATCH)` foi movida para o topo do "Prior amendment
  history"; adicionados os "Deferred TODOs" (infra + bootstrap dos 2 repos).
- **ESLint / testes: N/A** — a entrega é Markdown (`.specify/memory/constitution.md`), não há
  código TypeScript/JavaScript. `/speckit-unit-tests`, `/speckit-component-tests` e
  `/speckit-test` permanecem N/A (Q9).
- `git status --porcelain`: só `M .specify/memory/constitution.md`. Untracked pré-existentes
  (`services/beach-center-bff-injection/`, `services/beach-center-bff-llm-engine/`,
  `tasks/007-.../`) não foram tocados. **AC-10 satisfeito.**

---

## Critérios de Aceite (formais)

> Base para `/speckit-validate` e `/speckit-test`. Como a task é doc-only, os ACs são
> **verificáveis por leitura** do `.specify/memory/constitution.md` após o `/speckit-implement`.
> Rastreiam o **DoD do enunciado da US01** (DoD-1..DoD-5) + as decisões C1–C5.

### AC-1 — Repositórios novos declarados no Princípio I *(DoD-1)*
- **Given** a Constituição na versão pós-implementação
- **When** um leitor abre o **Princípio I (Fronteiras do Ecossistema)**
- **Then** a lista **Backend (BE - services)** contém `services/beach-center-bff-injection` e
  `services/beach-center-bff-llm-engine`, cada um com uma frase de **responsabilidade principal**
  fiel ao enunciado (ingestão/upload de comprovante; worker de validação visual assíncrona por IA).

### AC-2 — Stacks declaradas *(DoD-2, C1)*
- **Given** o texto novo
- **When** o leitor procura a stack de cada serviço novo
- **Then** está escrito que `beach-center-bff-injection` é **Node.js + TypeScript** e que
  `beach-center-bff-llm-engine` é **Node.js + TypeScript + Moondream + Docker**
- **And** ambos são descritos como **microsserviços puros** (não BFF), e o texto **não** chama
  `injection` de "Backend for Frontend".

### AC-3 — Novo Princípio VI existe e está numerado *(DoD-1, C2)*
- **Given** a seção "Core Principles"
- **When** o leitor percorre os princípios
- **Then** existe um **`### VI.`** com título sobre arquitetura orientada a eventos / validação
  de comprovantes, posicionado **após o Princípio V** e **antes** da seção "Comandos de
  Desenvolvimento".

### AC-4 — Fluxo assíncrono REST → MinIO → Fila → IA mapeado *(DoD-3)*
- **Given** o Princípio VI
- **When** o leitor procura o fluxo de ponta a ponta
- **Then** há um **bloco sequencial** (numerado ou ASCII) com os 8 passos: (1) Front→REST→injection,
  (2) injection→MinIO, (3) injection→agendamentos (status "Em Análise" + URL), (4) injection→Fila,
  (5) llm-engine←Fila, (6) llm-engine→MinIO (download), (7) llm-engine→inferência Moondream,
  (8) llm-engine→agendamentos (rejeita+libera quadra | mantém em análise)
- **And** está dito em texto que **o front não aguarda a IA** (resposta imediata de `injection`).

### AC-5 — Comunicação `injection` ↔ `llm-engine` é só mensageria *(DoD-4)*
- **Given** o Princípio VI
- **When** o leitor procura como os dois serviços se falam
- **Then** há uma regra **normativa** (`MUST` / `MUST NOT`) afirmando que a comunicação entre
  `beach-center-bff-injection` e `beach-center-bff-llm-engine` ocorre **exclusivamente por fila**
  (RabbitMQ **ou** BullMQ/Redis) e **nunca** por chamada REST síncrona.

### AC-6 — MinIO documentado como substituto do Google Drive por rate limit *(DoD-5, C-storage)*
- **Given** o Princípio VI
- **When** o leitor procura a decisão de storage
- **Then** está escrito que o **MinIO** (S3-compatible, self-hosted) é o **storage padrão** de
  comprovantes e que **substitui o Google Drive**
- **And** a **justificativa registrada** é o **rate limit** da API do Google Drive.

### AC-7 — Fronteira de dados preservada (Princípio I / C5)
- **Given** o Princípio VI
- **When** o leitor procura como `injection` e `llm-engine` gravam o status do agendamento
- **Then** está explícito que **ambos usam a REST interna (`x-api-key`) do
  `beach-center-bff-agendamentos`** e que **nenhum dos dois escreve diretamente** na coleção de
  reservas
- **And** o texto **não** introduz um banco de dados compartilhado nem uma coleção duplicada de
  status.

### AC-8 — Emenda registrada conforme a Governança (C2)
- **Given** o arquivo pós-implementação
- **When** o leitor abre o **Sync Impact Report** (topo) e a **linha de versão** (rodapé)
- **Then** o Sync Impact Report registra `1.3.1 → 1.4.0`, rationale **MINOR** (novo princípio),
  `Modified principles: I`, `Added sections: Princípio VI`, e preserva o histórico anterior
- **And** o rodapé lê `**Version**: 1.4.0 | **Ratified**: 2026-08-30 | **Last Amended**: 2026-09-10`.

### AC-9 — Conformidade dos serviços novos com II e III afirmada
- **Given** o Princípio VI
- **When** o leitor procura as obrigações de qualidade dos 2 repos
- **Then** o texto afirma que `injection` e `llm-engine` (Node.js/TypeScript) **seguem** o
  Princípio II (Hexagonal) e o Princípio III (ESLint + cobertura ≥ 80%), e que o consumo de fila
  entra pela camada de `controllers` de evento
- **And** **não** concede a nenhum dos dois a isenção de testes que o `beach-center-whatsapp` tem.

### AC-10 — Escopo mínimo respeitado (C3)
- **Given** o resultado do `/speckit-implement`
- **When** se roda `git status --porcelain`
- **Then** o **único** arquivo modificado é `.specify/memory/constitution.md`
- **And** `beach-center-server/docker-compose.dev.yml`, os repos `injection`/`llm-engine` e
  `beach-center-documentation/` **não** foram alterados.

### AC-11 — Nota de mapeamento com a máquina de estados da 006a (Q6)
- **Given** o Princípio VI
- **When** o leitor procura como "Em Análise" / "Rejeitado" se relacionam com o resto do sistema
- **Then** há uma **nota não-normativa** mapeando "Em Análise" ≈ `waiting_approve` e "Documento
  Inválido / Rejeitado" ≈ `rejected` (+ libera a quadra), referenciando a task 006a.

### AC-12 — Coerência interna do documento
- **Given** o documento inteiro pós-implementação
- **When** se procura por contagens/listas de princípios ("cinco princípios", "Princípios I a V",
  etc.) e por cross-references
- **Then** nenhuma passagem fica factualmente errada por causa da adição do Princípio VI
- **And** o Markdown continua válido (níveis de heading, blocos de código fechados, tabela do
  rodapé intacta).

---

## Rastreabilidade DoD → AC

| DoD (enunciado US01) | Critério(s) de Aceite |
|---|---|
| Arquivo de arquitetura atualizado com a descrição exata dos 2 repos | AC-1, AC-3 |
| Stacks de tecnologia declaradas | AC-2 |
| Fluxo assíncrono (REST → MinIO → Fila → IA) explicado e mapeado | AC-4 |
| Comunicação Injection ↔ LLM Engine estritamente por fila (não REST síncrono) | AC-5 |
| MinIO documentado como substituto do Google Drive (rate limit) | AC-6 |
| — (decisões C1–C5 / governança) | AC-2, AC-7, AC-8, AC-9, AC-10, AC-11, AC-12 |

---

## Próximo passo

`/speckit-implement` — aplicar E1–E5 (e avaliar E6) em `.specify/memory/constitution.md`.
Depois: `/speckit-validate` → `/speckit-complete`. (`/speckit-unit-tests`,
`/speckit-component-tests`, `/speckit-test` = **N/A** — doc-only.)
