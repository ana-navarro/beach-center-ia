# Task 008 — US02: Desacoplamento do Módulo de Pagamentos (upload de comprovante → `beach-center-bff-injection`)

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> **Depende da [[task-007-constitution-injection-llm-engine]]** (Princípio VI já ratificado —
> v1.4.0). Perguntas bloqueadoras **em aberto** — ver seção "Perguntas em aberto".

## Título

Extrair de `beach-center-bff-pagamentos` toda a lógica de **recebimento, validação e
armazenamento** do arquivo de comprovante de pagamento e migrá-la para o novo
`beach-center-bff-injection`, que passa a: validar o arquivo (formato + tamanho), subir para o
**MinIO**, marcar o agendamento como **"Em Análise"** (via REST interna do `agendamentos`) e
**publicar um evento na fila** para o `beach-center-bff-llm-engine` processar depois (US03+).
**Front-end não é alterado** — a nova API fica "preparada" para uso futuro.

## Contexto de negócio e técnico

- **Objetivo**: SRP. `pagamentos` cuida só de dados transacionais (valores, status lógico,
  gateway Getnet, webhook). O I/O pesado de arquivo (binário, imagem, storage) sai de lá.
- Prepara o terreno para a validação por IA (visão computacional) **sem gargalo síncrono** — o
  `injection` responde rápido e o `llm-engine` processa a fila em background (Princípio VI).

### Estado atual do código (levantado nesta task)

**O fluxo atual NÃO usa `multipart/form-data`.** O comprovante chega como **JSON com base64**:

| Item | Como é hoje |
|---|---|
| Rotas (em `beach-center-bff-pagamentos/src/applications/routes/routes.ts`) | `POST /transaction-history` (autenticado, Firebase) e `POST /public/transaction-history` (público, via `public_reserve_token` de 32 chars). Ambas → `TransactionHistoryController.create` → `SubmitManualPaymentUsecase`. |
| DTO (`create-manual-payment.dto.ts`) | JSON: `reserve_id`, `reserve_number`, `amount` (∈ {80,120,160}), `duration_hours` (∈ {1,2,3}), `user {name,email,phone}`, `slots[]` (1–3), `proof_file { file_name, mime_type ∈ [image/jpeg, image/png, application/pdf], base64 (máx ~7 MB de string ≈ "5 MB") }`, `public_reserve_token?`. |
| Storage | **Google Drive** (`src/infra/adapters/file-storage/google-drive-file-storage.adapter.ts`, via `fetch` + JWT service-account; `GOOGLE_DRIVE_*` env), com **fallback base64 no próprio documento** (`IStoredProof.storage: "google_drive" | "database"`). **Não há `multer`/`busboy`/`aws-sdk`.** |
| `SubmitManualPaymentUsecase` (fluxo) | 1) lê a reserva no `agendamentos` (REST `IReadReservePort`); 2) exige `reserve.status === "pending"`; autoriza por `public_reserve_token` **ou** e-mail do requester == `reserve.email`; 3) rejeita se já existe `manual_payment` para a reserva; 4) reconcilia `amount`/`slots`/`duration` com a reserva (tabela `{1:80,2:120,3:160}`); 5) **`PATCH /reservas/:id/status → waiting_approve`** no `agendamentos` (REST — **é essa transição que gera o `number`/protocolo**, task 006a); 6) `fileStoragePort.uploadProof(...)`; 7) cria o registro na coleção **`manual_payments`**. Compensação: se 6/7 falham → reserva volta a `rejected`. |
| Coleção própria do `pagamentos` | `manual_payments` (`IManualPaymentRepositoryPort`): `{ reserve_id, reserve_number, amount, duration_hours, status: "pending"\|"approved"\|"rejected", user, slots, proof: IStoredProof, admin_note?, reviewed_at? }`. |
| **Fluxo de revisão do admin (fica no `pagamentos`)** | `GET /transaction-history` (list), `GET /transaction-history/:id` (read), `GET /transaction-history/:id/proof` (**redirect** para `proof.view_url` do Drive **ou** serve o base64), `PATCH /transaction-history/:id/review` (`approved`/`rejected` → `ReviewManualPaymentUsecase`: atualiza `reserve.status` + `payment_method: "PIX"` via REST do `agendamentos`, marca o `manual_payment`). Tudo isso **lê da coleção `manual_payments` + serve o arquivo**. |
| Front (`beach-center-app`) | `src/private/services/payments/manual-payment.service.ts` faz `fileToBase64(file)` e `POST` JSON para `/transaction-history` ou `/public/transaction-history`. **Não muda nesta task.** |
| Comprovante de `ranking_agendamento` | Fica em `beach-center-bff-agendamentos`: `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/comprovante` (público, QR code) → `AttachProofRankingAgendamentoUsecase` → também **Google Drive** (`GOOGLE_DRIVE_*` no env do `agendamentos`) + fallback base64. **Não é o "módulo de pagamentos".** |
| Infra (`beach-center-server/docker-compose.dev.yml`) | **Não há MinIO, RabbitMQ nem Redis.** Não há serviço `injection` nem `llm-engine`. O Princípio VI registrou a infra como *Deferred TODO*. |
| `beach-center-bff-injection` | **Repo vazio** — só `package.json` bootstrap (`"type": "commonjs"`, sem deps, sem código, sem TS/ESLint/Jest). |

### Alinhamento com o Princípio VI (Constituição v1.4.0 — task 007)

O Princípio VI já normatiza:
- `injection` recebe upload REST, valida estático (**PDF/JPG/PNG**, **≤ 5 MB**), sobe pro **MinIO** (S3 SDK), grava status **"Em Análise"** + URL **via REST interna (`x-api-key`) do `agendamentos`**, publica `{ id_agendamento, url_arquivo }` na fila, responde rápido.
- Comunicação `injection ↔ llm-engine` **só por fila** (RabbitMQ ou BullMQ/Redis), nunca REST.
- `injection` **não** escreve direto na base de reservas nem mantém coleção paralela de status.
- MinIO **substitui** o Google Drive (rate limit).
- Nota: "Em Análise" ≈ `waiting_approve`; "Rejeitado" ≈ `rejected` + libera quadra.

**Tensão a resolver:** o texto da US02 fala em `multipart/form-data` (o fluxo atual é JSON+base64),
em "Atualização de Banco de Dados" pelo `injection` (o Princípio VI diz REST interna, sem banco
próprio), e em manter a revisão do admin no `pagamentos` (que hoje depende da coleção
`manual_payments` + do arquivo). Ver perguntas.

## Serviço(s) alvo (Princípio I)

| Repositório | Papel nesta task | Justificativa |
|---|---|---|
| **`services/beach-center-bff-injection`** (novo) | **Criação** da rota de ingestão de comprovante + validação estática + upload MinIO + marcação "Em Análise" (REST `agendamentos`) + publish na fila. Bootstrap TS + hexagonal + ESLint + Jest (Princípios II/III) — **escopo a confirmar**. | É o novo dono do I/O de comprovante (Princípio VI). |
| **`services/beach-center-bff-pagamentos`** | **Remoção**: rota(s) de submissão de comprovante (`POST /transaction-history` + `/public/...` — só a parte de `create`), `SubmitManualPaymentUsecase`, `IFileStoragePort` + `google-drive-file-storage.adapter`, DTO de submissão, env `GOOGLE_DRIVE_*`. **A definir**: o que acontece com `manual_payments` + fluxo de revisão. | US02: tirar o I/O de arquivo de lá. |
| ~~`beach-center-server`~~ | **A confirmar** — MinIO + RabbitMQ/Redis no `docker-compose.dev` (Princípio IV) pode ser desta task ou de uma task de infra separada. | Sem infra, o DoD "salvo no MinIO" / "publicado na fila" não é verificável. |
| **`beach-center-ia`** (root) | Bump do gitlink do submódulo `services/beach-center-bff-injection` (hoje untracked). | O root referencia os serviços como submódulos. |
| ~~`beach-center-app`~~ | **NÃO tocado** (DoD explícito). O front continua chamando a rota velha do `pagamentos` até uma task de front futura repontar. | — |
| **`beach-center-bff-agendamentos`** | **Afetado (C1/C3)** — `reserva` ganha `proof_key?: string` (model + schema); `PATCH /reservas/:id/payment` (ou rota equivalente) passa a aceitar `proof_key` para o `injection` gravar a referência do arquivo via REST interna. Ranking `attach-proof` **fora de escopo** (Q6). | Dono da reserva; já expõe as rotas internas `x-api-key`. |
| ~~`beach-center-bff-llm-engine`~~ | **NÃO tocado** — é consumidor da fila (US03+). | — |

## Regras de negócio conhecidas

1. Validação estática no `injection`: extensões `.png/.jpg/.jpeg/.pdf`, tamanho **≤ 5 MB** → **HTTP 400** com mensagem clara se falhar.
2. Nome do arquivo no MinIO = **UUID** (+ extensão). Retorna URL/path.
3. Após o upload: marca o agendamento como **"Pendente / Em Análise"** e associa a URL do MinIO.
4. Logo após o banco confirmar: **publica na fila** payload com no mínimo `agendamento_id`, `usuario_id`, `file_url`, `timestamp`.
5. `pagamentos` fica **livre** de manipulação de arquivo físico (upload/download de comprovante).
6. Front-end **não muda**.
7. (herança 006a) A submissão de comprovante move a reserva `pending → waiting_approve` — é o que gera o protocolo.
8. (herança 006a) Reembolso sempre por contato direto usuário↔admin — nada automático.

## Impacto arquitetural previsto (Arquitetura Hexagonal — Princípio II)

### `beach-center-bff-injection/src/` (NOVO — bootstrap TS + estrutura hexagonal)
- `applications/routes/` — `POST /api/v1/agendamentos/comprovantes` (nome a confirmar no plano).
- `applications/middlewares/` — validação estática do comprovante (decodifica base64 → checa
  `mime_type` + tamanho ≤ 5 MB) → **400**; `internalApiKeyMiddleware` (`x-api-key` — quem chama é
  o proxy do `pagamentos`, C5).
- `applications/controllers/comprovante/upload/` — recebe a requisição, chama o usecase.
- `applications/dto/` — `inject-comprovante.dto.ts` — **JSON + base64** (`reserve_id`,
  `reserve_number`, `amount`, `duration_hours`, `user`, `slots`, `proof_file { file_name,
  mime_type, base64 }`, `public_reserve_token?`, `usuario_id?`) — espelha o DTO atual do
  `pagamentos` (C2).
- `domain/models/` — `comprovante.model.ts` (dados do arquivo + agendamento + resultado).
- `domain/usecases/comprovante/inject-comprovante/` — **regra de negócio**: valida contexto
  (reserva existe e está `pending`; autorização por `public_reserve_token` ou e-mail; reconciliação
  amount/slots/duration), orquestra: `put-object` no MinIO → `PATCH status waiting_approve` +
  `proof_key` no `agendamentos` → `publish` na fila. Compensação se algum passo falhar.
- `domain/ports/output/` — `IObjectStoragePort` (put-object), `IReservaClientPort` (read reserva +
  marca `waiting_approve` + `proof_key` via REST), `IValidationQueuePort` (publish).
- `infra/adapters/object-storage/put-object/` — **AWS S3 SDK v3** (`@aws-sdk/client-s3`) apontando
  para o endpoint do MinIO (um adapter por verbo).
- `infra/adapters/reserva-client/{read,mark-in-review}/` — `axios` + `x-api-key` → `agendamentos`.
- `infra/adapters/validation-queue/publish/` — **RabbitMQ (`amqplib`)** (default Q10).
- `infra/schemas/` — **nenhum** (`injection` é stateless — C3).
- `config/` — `env.ts` (`PORT`, `MINIO_ENDPOINT`/`MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY`/
  `MINIO_BUCKET`/`MINIO_PUBLIC_URL`, `RABBITMQ_URL`/`RABBITMQ_QUEUE`, `AGENDAMENTOS_API_URL`,
  `AGENDAMENTOS_INTERNAL_API_KEY`, `INJECTION_INTERNAL_API_KEY`, `FIREBASE_*`?), `container.ts`.
- Raiz: `tsconfig.json` (`strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess` — como
  os outros serviços), `eslint.config.mjs` (flat), `jest.config` + `ts-jest` (coverage global 80%),
  `main.ts`/`index.ts`, `package.json` (converter `commonjs` → TS), `.env.example`.

### `beach-center-bff-pagamentos/src/` (REMOÇÃO + PROXY + REESCRITA da revisão — C3/C5)
- `applications/routes/routes.ts` — `POST /transaction-history` e `POST /public/transaction-history`
  **viram proxies finos** (repassam para `POST {INJECTION_API_URL}/agendamentos/comprovantes` com
  `x-api-key`). `GET /transaction-history`, `GET /:id`, `GET /:id/proof`, `PATCH /:id/review` —
  **mantidos**, mas reescritos para ler a reserva do `agendamentos`.
- `applications/controllers/transaction-history/transaction-history.controller.ts` — `create()` vira
  chamada ao proxy; `list`/`read` passam a listar/ler reservas `waiting_approve` via REST do
  `agendamentos`; `proof` **redireciona** para `${MINIO_PUBLIC_URL}/${bucket}/${proof_key}` (da
  reserva); `review` mantém a lógica (status + `payment_method`), remove a escrita em `manual_payments`.
- `applications/dto/create-manual-payment.dto.ts` — **remover** (validação passa a ser no `injection`).
- `domain/usecases/manual-payment/submit-manual-payment.usecase.ts` — **remover**.
- `domain/usecases/manual-payment/{query,review}-manual-payments.usecase.ts` — reescrever para a
  fonte `agendamentos` (sem `IManualPaymentRepositoryPort`).
- `domain/ports/output/file-storage.port.ts` + `infra/adapters/file-storage/google-drive-file-storage.adapter.ts` (+ `.spec`) — **remover**.
- `domain/ports/output/manual-payment-repository.port.ts` + `infra/adapters/.../manual-payment-*` — **remover** (coleção descontinuada).
- `domain/models/manual-payment.model.ts` — enxugar para o que a revisão precisa (view derivada da reserva).
- `config/env.ts` — remover `googleDrive.*`; `+ INJECTION_API_URL`, `INJECTION_INTERNAL_API_KEY`, `MINIO_PUBLIC_URL`, `MINIO_BUCKET`.
- `config/container.ts` — remover wiring de `submitManualPayment` + `fileStoragePort` + `manualPaymentRepository`; `+` client de proxy do `injection`.
- Testes: remover/reescrever `submit-manual-payment.usecase.spec.ts`, `google-drive-file-storage.adapter.spec.ts`, specs de `manual-payment-repository`, e ajustar specs de rota/controller/container.
- **Nota (task futura):** migração dos comprovantes legados do Drive → MinIO e drop físico da coleção `manual_payments`.

### `beach-center-server/` (C1)
- `docker-compose.dev.yml` — **`minio`** (+ um passo de init do bucket), **`rabbitmq`** (com painel de management), e **`injection`** (Node/TS, hot-reload por volume — Princípio IV) + volume `injection_node_modules`. Envs de `pagamentos` e `agendamentos` ganham `INJECTION_API_URL` / `MINIO_*` conforme necessário.
- `.env.dev.example` — `MINIO_*`, `RABBITMQ_*`, `INJECTION_INTERNAL_API_KEY`.

### `beach-center-bff-agendamentos/src/` (C3 — mínimo)
- `domain/models/reserva.model.ts` + `infra/schemas/reserva.schema.ts` — `+ proof_key?: string` (object key do comprovante no MinIO).
- `applications/dto/update-payment-metadata.dto.ts` + `domain/ports/*` + adapter de update — aceitar `proof_key` no `PATCH /reservas/:id/payment` (reuso; sem rota nova). O `injection` chama `PATCH /reservas/:id/status` (→ `waiting_approve`) **e** `PATCH /reservas/:id/payment` (`proof_key`).
- Docs (`beach-center-documentation/beach-center-bff-agendamentos/reserva.md`) atualizadas no `/speckit-documentation`.

## Decisões confirmadas (rodada 1 — `/speckit-task`)

| # | Pergunta | Decisão |
|---|---|---|
| **C1** | Escopo além da feature (repo vazio + sem infra) | **Tudo junto na US02**: (1) **bootstrap** do `beach-center-bff-injection` em **TypeScript + Arquitetura Hexagonal** (Princípio II) + **ESLint flat + Jest/ts-jest** com cobertura ≥ 80% (Princípio III) + Express; (2) **infra** no `beach-center-server/docker-compose.dev.yml` — `minio` + inicialização do bucket, `rabbitmq`, e o serviço `injection` (Node/TS com hot-reload, Princípio IV) + volume de `node_modules` + envs no `.env.dev.example`; (3) a rota de ingestão + a limpeza no `pagamentos`. Repos afetados: `injection` (novo), `pagamentos` (proxy + reescrita da revisão), `beach-center-server` (infra), `beach-center-bff-agendamentos` (campo `proof_key`), `beach-center-ia` (gitlink do submódulo). |
| **C2** | Contrato da nova rota | **JSON + base64** — espelha o contrato atual (`proof_file: { file_name, mime_type, base64 }`). **Sem `multer`.** Validação estática: decodifica o base64 e checa `mime_type` ∈ `image/jpeg`/`image/png`/`application/pdf` **e** tamanho decodificado **≤ 5 MB** → **HTTP 400** com mensagem clara. Quando o front migrar (task futura) troca só a URL, não o corpo. |
| **C3** | `manual_payments` + revisão do admin | **`injection` stateless** (sem banco/schema próprio — alinhado ao Princípio VI). Grava **via REST interna (`x-api-key`) do `agendamentos`**: `reserva.status → waiting_approve` (reusa `PATCH /reservas/:id/status`) **+** `reserva.proof_key` (campo novo na reserva). A coleção **`manual_payments` é descontinuada**. O **painel de revisão** (`GET /transaction-history`, `/:id`, `/:id/proof`, `PATCH /:id/review`) **fica no `pagamentos`** mas passa a **ler as reservas `waiting_approve` do `agendamentos`** (via REST) em vez da coleção; `/:id/proof` **redireciona** para `${MINIO_PUBLIC_URL}/${bucket}/${proof_key}`; `PATCH /:id/review` mantém a lógica (aprovar/rejeitar → status da reserva + `payment_method`), só remove a escrita em `manual_payments`. **Migração** dos comprovantes legados (Drive → MinIO) e drop da coleção antiga = **nota para task futura**. |
| **C5** | Rota antiga (front não muda) | **Proxy fino**: `POST /transaction-history` e `POST /public/transaction-history` continuam no `pagamentos`, mas **só repassam** a requisição para `POST {INJECTION_API_URL}/agendamentos/comprovantes` (com `x-api-key`), **sem nenhuma lógica de arquivo/storage** (cumpre o SRP do DoD-1). O front não percebe. As rotas-proxy são removidas numa **task de front futura** que aponta direto para o `injection`. |

## Perguntas menores em aberto (defaults propostos — fechar no `/speckit-plan`)

> As 4 bloqueadoras foram fechadas acima (C1, C2, C3, C5). Estas não bloqueiam o plano.

6. **Escopo: só reserva comum, ou também `ranking_agendamento`?** Default: **só reserva comum**
   (o `attach-proof` de ranking está no `agendamentos`, não no "módulo de pagamentos"; migrar
   depois).

7. **`usuario_id` no payload da fila.** O fluxo não tem `usuario_id` hoje (só `user{name,email,
   phone}` + token de link público). Default: **`usuario_id` = uid do Firebase quando
   autenticado; omitido/`null` no fluxo público por link**. (Ou usar o e-mail como id.)

8. **Como o `injection` grava `proof_key` + "Em Análise" no `agendamentos`.** Default: **reusa
   `PATCH /reservas/:id/status` → `waiting_approve`** (já existe, já gera o protocolo — 006a) +
   **estende `PATCH /reservas/:id/payment`** (ou o DTO `update-payment-metadata`) com um campo
   `proof_key`. Alternativa: rota nova `PATCH /reservas/:id/proof`. Decidir no `/speckit-plan` —
   preferir reuso. O `agendamentos` ganha `reserva.proof_key?: string` (model + schema).

9. **Formato da `file_url` / `proof_key`.** Default: **guardar a object key** (ex.:
   `comprovantes/<uuid>.<ext>`); a URL base do MinIO vem de env (`MINIO_PUBLIC_URL`) e quem
   precisar (painel de revisão, `llm-engine`) monta `${MINIO_PUBLIC_URL}/${bucket}/${key}`. **Não**
   usar URL pré-assinada (expiraria antes da IA processar). O `file_url` no payload da fila = a
   URL completa montada (conveniência do consumer).

10. **Fila: RabbitMQ ou BullMQ/Redis?** Default: **RabbitMQ + `amqplib`** — não há Redis no
    ecossistema; RabbitMQ é mensageria "de verdade" e simples de subir no `docker-compose.dev`.
    Confirmar no `/speckit-plan`. Nome da fila/exchange e durabilidade a definir no plano
    (provável: exchange direct `comprovantes` / routing key `comprovante.validar`, fila durável).

11. **Payload da fila.** Default: `{ agendamento_id, usuario_id, email, file_url, proof_key,
    bucket, mime_type, timestamp }` — cobre o mínimo do DoD (`agendamento_id`, `usuario_id`,
    `file_url`, `timestamp`) + campos úteis pro `llm-engine`. `usuario_id` = uid Firebase quando
    autenticado, `null` no fluxo público (Q7).

12. **Ciclo Speckt.** Repos afetados: `beach-center-bff-injection` (novo — testes de verdade,
    cobertura ≥ 80%) + `beach-center-bff-pagamentos` (proxy + reescrita da revisão + remoção de
    testes órfãos) + `beach-center-bff-agendamentos` (`proof_key` — model/schema/dto/port) +
    `beach-center-server` (infra — sem testes) + `beach-center-ia` (gitlink). `/speckit-unit-tests`
    roda nos 3 serviços TS; `/speckit-component-tests` = **N/A** (front não muda). Confirmar.

## Notas

- Atualiza a memória `[[sistema-pagamento-futuro]]` e `[[task-007-constitution-injection-llm-engine]]`.
- Esta é a **US02** de uma série (US01 = task 007 / constitution; US03+ prováveis = consumer da
  fila no `llm-engine`, migração do front, painel de revisão com MinIO).
- O `beach-center-bff-agendamentos` já expõe `PATCH /reservas/:id/status` e
  `PATCH /reservas/:id/payment` com `x-api-key` (`adminOrInternalApiKey`) — o `pagamentos` já usa
  isso hoje; o `injection` reusaria o mesmo caminho.
- `beach-center-bff-pagamentos` está na `main` (pós-006a). `beach-center-bff-agendamentos` idem
  (pós-006a/006b). Nenhuma branch aberta.

## Próximo passo

Perguntas bloqueadoras (C1, C2, C3, C5) **fechadas**. Rodar **`/speckit-plan`** — ele confirma os
defaults Q6–Q12, decide os detalhes (nome da rota, forma de gravar `proof_key`, nomes de
fila/bucket, estrutura exata do bootstrap) e gera o checklist + Critérios de Aceite cobrindo o DoD
da US02.
