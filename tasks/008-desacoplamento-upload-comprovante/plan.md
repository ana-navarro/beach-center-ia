# Plan — Task 008 (US02): Desacoplamento do upload de comprovante → `beach-center-bff-injection`

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`.
> Bloqueadoras (C1, C2, C3, C5) fechadas no `/speckit-task`. Defaults Q6–Q12 confirmados abaixo.

---

## Contexto Técnico

### Serviço(s) alvo e justificativa (Princípio I)

| Repositório | Papel | Justificativa da fronteira |
|---|---|---|
| **`services/beach-center-bff-injection`** (NOVO) | Bootstrap TS + hexagonal; rota `POST /api/v1/agendamentos/comprovantes`; validação estática; upload MinIO; marca `waiting_approve` + `proof_key` via REST do `agendamentos`; publica na fila RabbitMQ. **Stateless** (sem banco). | Dono do I/O de comprovante (Princípio VI). |
| **`services/beach-center-bff-pagamentos`** | Rotas de submissão viram **proxy fino** → `injection`. Remove `SubmitManualPaymentUsecase`, `GoogleDriveFileStorageAdapter`, `IFileStoragePort`, `IManualPaymentRepositoryPort` + coleção `manual_payments`. Reescreve `list`/`read`/`proof`/`review` para a fonte `agendamentos`. | Fica só com dados transacionais + a **decisão** de aprovar/rejeitar (status lógico do pagamento). |
| **`services/beach-center-bff-agendamentos`** | `reserva` ganha `proof_key?` e `review_note?`; `PATCH /reservas/:id/payment` passa a aceitá-los. | Dono único da coleção de reservas (Princípio VI — `injection` não escreve direto). |
| **`beach-center-server`** | `docker-compose.dev.yml`: serviços `minio` (+ init do bucket), `rabbitmq`, `injection` (hot-reload). `.env.dev.example`. | Princípio IV — infra reproduzível. |
| **`beach-center-ia`** (root) | Gitlink do submódulo `services/beach-center-bff-injection` (hoje untracked). | Root referencia os serviços como submódulos. |
| ~~`beach-center-app`~~ | **NÃO tocado** (DoD-6). | Front repontará numa task futura. |
| ~~`beach-center-bff-llm-engine`~~ | **NÃO tocado** — consumidor da fila (US03+). | — |

### Stack e bibliotecas

- **`injection`** (novo): Node 20 + **TypeScript** (mesmo `tsconfig` dos outros: `strict`,
  `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`), **Express 5**, **yup**, **axios**,
  **cors**, **dotenv**, **`@aws-sdk/client-s3`** (endpoint MinIO), **`amqplib`** (RabbitMQ).
  UUID via `crypto.randomUUID()` (nativo — sem dep). **Sem** `mongoose`/`firebase-admin`
  (stateless; auth de entrada é `x-api-key`). ESLint flat + `typescript-eslint` + regras de
  isolamento hexagonal (copiadas de `campeonatos`). **Jest + ts-jest**, `coverageThreshold`
  global 80%.
- **`pagamentos`**: sem libs novas — remove o uso de `googleapis`/`fetch` p/ Drive. `+ axios`
  já existe (proxy + client do `agendamentos`).
- **`agendamentos`**: sem libs novas.
- **infra**: imagens `minio/minio`, `minio/mc` (init do bucket), `rabbitmq:3-management`.

### Decisões de planejamento (fecham Q6–Q12 + detalhes)

| # | Decisão |
|---|---|
| **DP1** | Rota no `injection`: **`POST /api/v1/agendamentos/comprovantes`** (nome do enunciado). Guard: **`internalApiKeyMiddleware`** (`INJECTION_INTERNAL_API_KEY`) — só o proxy do `pagamentos` chama. |
| **DP2** | **Q6** — escopo = **só reserva comum**. `ranking_agendamento` (`attach-proof` no `agendamentos`) fica para task futura. |
| **DP3** | **Contrato (C2)**: JSON. Corpo = o mesmo do DTO atual do `pagamentos` + `usuario_id?`/`requester_email?` (injetados pelo proxy): `{ reserve_id, reserve_number, amount, duration_hours, user{name,email,phone}, slots[], proof_file{file_name, mime_type, base64}, public_reserve_token?, usuario_id?, requester_email? }`. |
| **DP4** | **Validação estática** (middleware, antes do usecase): `mime_type` ∈ `image/jpeg`/`image/png`/`application/pdf`; `Buffer.from(base64,'base64').byteLength` ≤ **5 MiB** (`5*1024*1024`); base64 não-vazio e decodificável. Falha → **HTTP 400** `{ message }` claro. |
| **DP5** | **Autorização da reserva** (no usecase, defense-in-depth): reserva existe **e** `status === 'pending'` (senão 409); acesso liberado por **`public_reserve_token` válido** (checado no `agendamentos`) **ou** `requester_email` == `reserve.email`. Espelha o `SubmitManualPaymentUsecase` atual. |
| **DP6** | **Reconciliação** (no usecase): `amount`/`duration_hours`/`slots` conferem com a reserva (tabela `{1:80,2:120,3:160}`) — igual ao atual. Divergência → 400. |
| **DP7** | **Gravação no `agendamentos`** (via REST `x-api-key`), nesta ordem: (1) `PATCH /reservas/:id/status` `{status:"waiting_approve"}` → resposta traz `reserve.number` (protocolo gerado — 006a); (2) `PATCH /reservas/:id/payment` `{ proof_key }`. |
| **DP8** | **`proof_key`** = `comprovantes/<uuid>.<ext>` (`ext` derivada do mime: `jpg`/`png`/`pdf`). Upload no MinIO com `ContentType: mime_type` no bucket `MINIO_BUCKET`. **Sem URL pré-assinada.** `file_url` (conveniência do consumer) = `${MINIO_PUBLIC_URL}/${MINIO_BUCKET}/${proof_key}`. |
| **DP9** | **Fila (Q10)** = **RabbitMQ + `amqplib`**. Fila durável **`comprovante.validar`** (default exchange, `sendToQueue`, `persistent: true`). Conexão = singleton lazy no adapter. Payload (Q11): `{ agendamento_id, usuario_id: string\|null, email, file_url, proof_key, bucket, mime_type, timestamp: ISO }`. Publicado **depois** do passo DP7. |
| **DP10** | **Compensação**: upload OK mas DP7 falha → best-effort `DeleteObject` no MinIO + erro 502. DP7 OK mas publish falha → compensa reserva para `pending` (mantém `number`, que é idempotente) + best-effort delete do objeto + erro 502. |
| **DP11** | **Proxy no `pagamentos`** (C5): `POST /transaction-history` e `POST /public/transaction-history` continuam; o controller autentica (Firebase p/ a privada; nada p/ a pública), monta `{ ...req.body, usuario_id?: req.authUser?.uid, requester_email?: req.authUser?.email }` e faz `POST ${INJECTION_API_URL}/agendamentos/comprovantes` com `x-api-key: INJECTION_INTERNAL_API_KEY`; repassa status + corpo da resposta. **Zero lógica de arquivo/storage.** |
| **DP12** | **Painel de revisão no `pagamentos`** passa a ler o `agendamentos`: `list` → `GET /reservas?status=waiting_approve` (+ `name` p/ busca); `read` → `GET /reservas/:id`; `proof` → **302** para `${MINIO_PUBLIC_URL}/${MINIO_BUCKET}/${reserve.proof_key}` (404 se sem `proof_key`); `review` → mantém `ReviewManualPaymentUsecase` (status + `payment_method:"PIX"` no approve) **sem** `manual_payments`; `admin_note` gravado via `PATCH /reservas/:id/payment` `{ review_note }`. |
| **DP13** | **`agendamentos`**: `reserva` ganha `proof_key?: string` e `review_note?: string` (model `IReserve`/`IReserveDocument`/`toDomainReserve` + schema + `IReservePaymentMetadataUpdate` + `updatePaymentMetadataDTO`). O adapter já faz `$set: data` — nada além do tipo. |
| **DP14** | **Ciclo Speckt (Q12)**: `/speckit-unit-tests` roda em `injection` (cobertura ≥ 80% — foco em usecase + adapters + controller + middleware) e `pagamentos` (ajuste/remoção de specs). `agendamentos` ganha specs dos campos novos. `/speckit-component-tests` = **N/A** (front não muda). `beach-center-server` = sem testes. |
| **DP15** | **Submódulo**: `beach-center-bff-injection` passa a ser rastreado como gitlink pelo root (como os demais em `services/`), com `main` própria. Bump do gitlink acontece no `/speckit-complete`. |

---

## Constitution Check

| Princípio | Resultado | Justificativa |
|---|---|---|
| **I — Fronteiras** | **CONFORME** | `injection` novo já está no Princípio I (task 007). `pagamentos` perde o I/O de arquivo (SRP). `injection` **não** escreve na base de reservas — usa a REST interna do `agendamentos` (DP7). Coleção `manual_payments` (estado paralelo) é **descontinuada** — alinhado ao Princípio VI. Comunicação `injection → llm-engine` = **só fila** (DP9), sem REST. |
| **II — Arquitetura Hexagonal** | **CONFORME** | `injection` nasce com a estrutura padrão (`applications`/`domain`/`infra`/`config`), ESLint com `no-restricted-imports` isolando as camadas. Regra de negócio (validação de contexto, reconciliação, orquestração, compensação) no **usecase**; MinIO/RabbitMQ/HTTP só em **adapters** (um por verbo: `put-object`, `delete-object`, `publish`, `read-reserva`, `patch-reserva-status`, `patch-reserva-payment`, `check-public-link`). `domain` fala com `infra` só por ports. `pagamentos`: o proxy e os clients do `agendamentos` são adapters; os usecases de revisão não acessam schema. |
| **III — Test-First e Qualidade** | **CONFORME** | `injection` com Jest + ts-jest + `coverageThreshold` global 80%; ESLint estrito (flat, `typescript-eslint`). Portas de infra 100% mockadas nos testes de domínio. `pagamentos` mantém o gate; specs órfãos removidos, novos para o proxy e a revisão via `agendamentos`. `@wip` não usado. |
| **IV — Infra Reproduzível** | **CONFORME** | `minio`, `rabbitmq` e `injection` entram no `docker-compose.dev.yml` com bind-mount + `nodemon --legacy-watch` (hot-reload) e volume de `node_modules`, seguindo o padrão `x-node-build`/`x-bff-*`. Bucket criado por um serviço `minio-init` (`minio/mc`). |
| **V — Ciclo Speckt** | **CONFORME** | Task → plan → implement → unit-tests → (component N/A) → validate → test → complete → documentation. |
| **VI — Orientada a Eventos p/ Comprovantes** | **CONFORME** | Implementa exatamente o Princípio VI: REST → MinIO → Fila; `injection` stateless gravando via REST do `agendamentos`; MinIO no lugar do Drive; front não aguarda IA (resposta imediata após publish). |

**Resultado: SEM VIOLAÇÕES.** Prosseguir para `/speckit-implement`.

> **Risco registrado (não é violação):** remover o `SubmitManualPaymentUsecase` e manter o proxy
> significa que o envio de comprovante em produção passa a depender do `injection` estar no ar
> (novo ponto de falha). Mitigação: o proxy propaga erros do `injection` com status/mensagem
> claros; `docker-compose` com `depends_on` + healthcheck.

---

## Mapa Arquitetural (Hexagonal)

### A) `services/beach-center-bff-injection` — NOVO

**Bootstrap (raiz do repo)**
- `package.json` — reescrever a partir do template de `campeonatos` (`type: commonjs`, scripts
  `dev`/`build`/`start`/`lint`/`lint:fix`/`test`/`test:coverage`) + deps `@aws-sdk/client-s3`,
  `amqplib`; devDeps `@types/amqplib`.
- `tsconfig.json` — cópia de `campeonatos` (exclui `*.spec.ts`).
- `eslint.config.mjs` — cópia de `campeonatos` (isolamento de camadas + specs relaxados).
- `jest.config.ts` — cópia de `campeonatos` (`coverageThreshold` 80, ignora `main.ts`, `@wip`).
- `.env.example` — todas as envs abaixo.
- `.gitignore` — `node_modules`, `dist`, `coverage`, `.env`.

**`config/`**
- `config/env.ts` — `loadEnv()` com cache: `NODE_ENV`, `PORT` (default 5005), `INJECTION_INTERNAL_API_KEY`, `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY`, `MINIO_ENDPOINT`, `MINIO_REGION` (default `us-east-1`), `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_BUCKET`, `MINIO_PUBLIC_URL`, `MINIO_FORCE_PATH_STYLE` (default `true`), `RABBITMQ_URL`, `RABBITMQ_QUEUE` (default `comprovante.validar`). Faltando obrigatória → erro claro no boot / na 1ª chamada (padrão dos outros clients).
- `config/container.ts` — instancia adapters + `InjectComprovanteUsecase`, exporta `container`.

**`domain/`**
- `domain/errors.ts` — cópia da hierarquia `DomainError`/`InvalidInputError`/`NotFoundError`/`ConflictError`/`ForbiddenError`/`UpstreamError(502)`.
- `domain/models/comprovante.model.ts` — `IComprovanteFile { file_name; mime_type; base64 }`, `IInjectComprovanteInput { reserve_id; reserve_number; amount; duration_hours: 1|2|3; user; slots[]; proof_file: IComprovanteFile; public_reserve_token?; usuario_id?; requester_email? }`, `IInjectComprovanteResult { agendamento_id; numero_protocolo; proof_key; file_url; queued: true }`, `IReserveView` (subset lido do `agendamentos`).
- `domain/models/reserve-slot.model.ts` — `IReserveSlot { id; date; start_time; end_time; court; unit }`.
- `domain/ports/input/comprovante.input-port.ts` — `IInjectComprovanteUseCase.execute(input): Promise<IInjectComprovanteResult>`.
- `domain/ports/output/reserva-client.port.ts` — `IReadReservaPort.execute(id): Promise<IReserveView | null>`, `ICheckPublicLinkAccessPort.execute(token, reserveId): Promise<boolean>`, `ISetReservaInReviewPort.execute(id): Promise<{ number: string }>` (→ status `waiting_approve`), `ISetReservaProofPort.execute(id, proofKey): Promise<void>`.
- `domain/ports/output/object-storage.port.ts` — `IPutObjectPort.execute({ key; body: Buffer; contentType }): Promise<void>`, `IDeleteObjectPort.execute(key): Promise<void>`.
- `domain/ports/output/validation-queue.port.ts` — `IPublishValidationEventPort.execute(payload): Promise<void>`.
- `domain/usecases/comprovante/inject-comprovante/inject-comprovante.usecase.ts` — orquestra DP5→DP6→(put-object)→DP7→DP9, com a compensação DP10. Lança `NotFoundError`/`ConflictError`/`ForbiddenError`/`InvalidInputError`/`UpstreamError`.
- `domain/usecases/shared/handle-usecase-error.ts` — cópia.
- `domain/usecases/shared/reconcile-reserve.ts` — helper puro da reconciliação DP6 (tabela de preço + comparação de slots).

**`infra/adapters/` (um por verbo)**
- `infra/adapters/object-storage/put-object/put-object.adapter.ts` — `@aws-sdk/client-s3` `PutObjectCommand` (endpoint MinIO, `forcePathStyle`).
- `infra/adapters/object-storage/delete-object/delete-object.adapter.ts` — `DeleteObjectCommand` (best-effort; loga e engole).
- `infra/adapters/reserva-client/read-reserva/read-reserva.adapter.ts` — `GET {AGENDAMENTOS_API_URL}/reservas/:id` (`x-api-key`); 404 → `null`.
- `infra/adapters/reserva-client/check-public-link/check-public-link.adapter.ts` — valida o token contra o `agendamentos` (espelha `check-public-link-authorization.adapter` do `pagamentos`).
- `infra/adapters/reserva-client/set-in-review/set-reserva-in-review.adapter.ts` — `PATCH {..}/reservas/:id/status` `{status:"waiting_approve"}`; devolve `{ number }`.
- `infra/adapters/reserva-client/set-proof/set-reserva-proof.adapter.ts` — `PATCH {..}/reservas/:id/payment` `{ proof_key }`.
- `infra/adapters/validation-queue/publish/publish-validation-event.adapter.ts` — `amqplib`: conexão singleton lazy, `assertQueue(RABBITMQ_QUEUE,{durable:true})`, `sendToQueue(Buffer.from(JSON.stringify(payload)),{persistent:true})`.
- `infra/shared/http-error.ts` (opcional) — normaliza erros axios em `UpstreamError`.

**`applications/`**
- `applications/routes/routes.ts` — `routes.use('/agendamentos', comprovanteRoute)`.
- `applications/routes/comprovante.route.ts` — `POST /comprovantes` → `internalApiKeyMiddleware` → `staticFileValidationMiddleware` → `uploadComprovanteController`.
- `applications/middlewares/internal-api-key.middleware.ts` — cópia (checa `INJECTION_INTERNAL_API_KEY`).
- `applications/middlewares/static-file-validation.middleware.ts` — DP4.
- `applications/dto/inject-comprovante.dto.ts` — yup (DP3), mime `oneOf` + `base64` obrigatório (o tamanho é checado no middleware, sobre bytes).
- `applications/controllers/comprovante/upload/upload-comprovante.controller.ts` — valida DTO, chama `container.injectComprovante.execute`, responde `202 Accepted` `{ message: "Comprovante recebido e em processamento", data: result }`.
- `applications/controllers/shared/handle-http-error.ts` — cópia.
- `src/main.ts` — express + cors + json (limite `10mb` p/ o base64) + `/api/v1` routes + healthcheck `GET /api/v1`. Sem `mongoose`.

### B) `services/beach-center-bff-pagamentos` — REMOÇÃO + PROXY + REESCRITA

**Remover**
- `domain/usecases/manual-payment/submit-manual-payment.usecase.ts` (+ `.spec`).
- `domain/ports/output/file-storage.port.ts`; `infra/adapters/file-storage/google-drive-file-storage.adapter.ts` (+ `.spec`).
- `domain/ports/output/manual-payment-repository.port.ts`; `infra/adapters/manual-payment/mongo-manual-payment-repository.adapter.ts` (+ `.spec`); `infra/schemas/manual-payment.schema.ts` (se existir).
- `applications/dto/create-manual-payment.dto.ts` (+ `.spec`).
- `domain/models/manual-payment.model.ts` — reduzir para `IManualPaymentView` (derivado da reserva: `{ id, reserve_number, amount, duration_hours, status, user, slots, proof_url?, admin_note?, reviewed_at? }`) + `IReviewManualPaymentInput`.
- `config/env.ts` — remover `googleDrive.*`.

**Adicionar / alterar**
- `config/env.ts` — `+ injectionApiUrl` (`INJECTION_API_URL`), `+ injectionInternalApiKey` (`INJECTION_INTERNAL_API_KEY`), `+ minioPublicUrl` (`MINIO_PUBLIC_URL`), `+ minioBucket` (`MINIO_BUCKET`).
- `domain/ports/output/injection-client.port.ts` — `IForwardComprovantePort.execute(body, headers): Promise<{ status: number; body: unknown }>`.
- `infra/adapters/injection/forward-comprovante.adapter.ts` — `axios.post(${INJECTION_API_URL}/agendamentos/comprovantes, body, { headers: { 'x-api-key': INJECTION_INTERNAL_API_KEY }, validateStatus: () => true })`.
- `domain/ports/output/reserve.port.ts` — `+ IListReservesForReviewPort.execute(filter): Promise<IReserveView[]>` (ou estende o read port). Adapter: `GET {AGENDAMENTOS_API_URL}/reservas?status=waiting_approve[&name=]`.
- `infra/adapters/reserve/list-reserves-for-review.adapter.ts` — novo.
- `domain/usecases/manual-payment/query-manual-payments.usecase.ts` — `ListManualPaymentsUsecase` → lista reservas `waiting_approve` (map p/ `IManualPaymentView`); `ReadManualPaymentUsecase` → lê reserva por id; `GetManualPaymentProofUsecase` → devolve `{ redirect_url }` a partir de `${MINIO_PUBLIC_URL}/${MINIO_BUCKET}/${reserve.proof_key}` (ou `NotFoundError` se sem `proof_key`).
- `domain/usecases/manual-payment/review-manual-payment.usecase.ts` — remove dependência de `IManualPaymentRepositoryPort`; lê a reserva; valida `status ∈ {pending, waiting_approve}`; `approved` → `updateReservePaymentMetadataPort {payment_method:"PIX", review_note?}` + `updateReserveStatusPort "approved"`; `rejected` → `updateReserveStatusPort "rejected"` (+ `review_note?`).
- `applications/controllers/transaction-history/transaction-history.controller.ts` — `create` → `forwardComprovantePort` (proxy DP11); `proof` → `res.redirect(302, result.redirect_url)`; `list`/`read`/`review` → usecases reescritos.
- `applications/routes/routes.ts` — inalterado nos paths (mantém `POST /transaction-history`, `/public/transaction-history`, `GET /transaction-history[...]`, `PATCH .../review`).
- `config/container.ts` — remove `submitManualPayment`, `fileStorageAdapter`, `manualPaymentRepositoryAdapter`; adiciona `forwardComprovanteAdapter`, `listReservesForReviewAdapter`; recompõe os usecases de query/review.

### C) `services/beach-center-bff-agendamentos` — MÍNIMO (DP13)

- `domain/models/reserva.model.ts` — `+ proof_key?: string | undefined` e `+ review_note?: string | undefined` nas 3 interfaces relevantes (`IReserve`, doc, create/update).
- `infra/schemas/reserva.schema.ts` — `proof_key: { type: String, required: false }`, `review_note: { type: String, required: false }`; mapear em `toDomainReserve`.
- `domain/ports/output/reserve-persistence.port.ts` — `IReservePaymentMetadataUpdate + proof_key?`, `+ review_note?`.
- `applications/dto/update-payment-metadata.dto.ts` — `+ proof_key: yup.string().optional()`, `+ review_note: yup.string().optional()`.
- (o adapter `update-reserve-payment-metadata` faz `$set: data` — sem alteração de código, só o tipo).
- Specs: `update-payment-metadata.dto` / `.adapter` / model — casos dos campos novos.

### D) `beach-center-server` — INFRA (C1)

- `docker-compose.dev.yml`:
  - `minio` — `image: minio/minio`, `command: server /data --console-address ":9001"`, portas `9000:9000` + `9001:9001`, volume `minio_data:/data`, env `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` (defaults dev), healthcheck.
  - `minio-init` — `image: minio/mc`, `depends_on: minio (healthy)`, `entrypoint` que faz `mc alias set` + `mc mb --ignore-existing local/comprovantes` + `mc anonymous set download local/comprovantes` (leitura pública p/ o painel/IA no dev).
  - `rabbitmq` — `image: rabbitmq:3-management`, portas `5672:5672` + `15672:15672`, healthcheck (`rabbitmq-diagnostics -q ping`).
  - `injection` — `<<: *node-build`, `command: *bff-command`, env `<<: *bff-env` + `PORT: "5005"`, `INJECTION_INTERNAL_API_KEY: dev-injection-key`, `AGENDAMENTOS_API_URL: http://agendamentos:5000/api/v1`, `AGENDAMENTOS_INTERNAL_API_KEY: dev-agendamentos-key`, `MINIO_ENDPOINT: http://minio:9000`, `MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY` (= root dev), `MINIO_BUCKET: comprovantes`, `MINIO_PUBLIC_URL: http://localhost:9000`, `MINIO_FORCE_PATH_STYLE: "true"`, `RABBITMQ_URL: amqp://rabbitmq:5672`, `RABBITMQ_QUEUE: comprovante.validar`; volumes bind-mount + `injection_node_modules`; `depends_on` mongo/firebase + `minio-init` (completed) + `rabbitmq` (healthy); portas `5005:5005`; healthcheck.
  - `pagamentos.environment` — `+ INJECTION_API_URL: http://injection:5005/api/v1`, `INJECTION_INTERNAL_API_KEY: dev-injection-key`, `MINIO_PUBLIC_URL: http://localhost:9000`, `MINIO_BUCKET: comprovantes`.
  - `volumes:` — `+ minio_data`, `+ injection_node_modules`.
  - `nginx` (se rotear por serviço) — rota `/injection` **não** exposta ao público (só o `pagamentos` fala com o `injection`). Confirmar no implement se o nginx tem bloco por serviço.
- `.env.dev.example` — comentário + `MINIO_*`, `RABBITMQ_*`, `INJECTION_INTERNAL_API_KEY`.

### E) `beach-center-ia` (root)

- Passar a rastrear `services/beach-center-bff-injection` como gitlink (o repo já tem `.git`); bump no `/speckit-complete`.
- `.claude`/docs: nenhum nesta etapa (docs da API = `/speckit-documentation`).

---

## Checklist de Implementação

> Ordem de dependência. Marcar `- [x]` no fim de cada item.

### Fase 1 — `agendamentos` (`proof_key` / `review_note`) — desbloqueia o `injection`
- [x] 1. `src/domain/models/reserva.model.ts` — `+ proof_key?`, `+ review_note?` nas interfaces.
- [x] 2. `src/infra/schemas/reserva.schema.ts` — campos + mapeamento em `toDomainReserve`.
- [x] 3. `src/domain/ports/output/reserve-persistence.port.ts` — `IReservePaymentMetadataUpdate + proof_key?, review_note?`.
- [x] 4. `src/applications/dto/update-payment-metadata.dto.ts` — `+ proof_key`, `+ review_note` (`yup.string().optional()`).

### Fase 2 — `beach-center-bff-injection` — bootstrap
- [x] 5. `package.json` (reescrever), `tsconfig.json`, `eslint.config.mjs`, `jest.config.ts`, `.gitignore`, `.env.example`.
- [x] 6. `src/domain/errors.ts`; `src/domain/usecases/shared/handle-usecase-error.ts`; `src/applications/controllers/shared/handle-http-error.ts`.
- [x] 7. `src/config/env.ts` (`loadEnv` + cache + validação).
- [x] 8. `src/main.ts` (express, cors, json 10mb, `/api/v1`, healthcheck).

### Fase 3 — `injection` — domínio
- [x] 9. `src/domain/models/reserve-slot.model.ts`, `src/domain/models/comprovante.model.ts`.
- [x] 10. `src/domain/ports/input/comprovante.input-port.ts`.
- [x] 11. `src/domain/ports/output/{reserva-client,object-storage,validation-queue}.port.ts`.
- [x] 12. `src/domain/usecases/shared/reconcile-reserve.ts`.
- [x] 13. `src/domain/usecases/comprovante/inject-comprovante/inject-comprovante.usecase.ts` (DP5–DP10).

### Fase 4 — `injection` — infra (adapters, um por verbo)
- [x] 14. `src/infra/adapters/object-storage/put-object/put-object.adapter.ts`.
- [x] 15. `src/infra/adapters/object-storage/delete-object/delete-object.adapter.ts`.
- [x] 16. `src/infra/adapters/reserva-client/read-reserva/read-reserva.adapter.ts`.
- [x] 17. `src/infra/adapters/reserva-client/check-public-link/check-public-link.adapter.ts`.
- [x] 18. `src/infra/adapters/reserva-client/set-in-review/set-reserva-in-review.adapter.ts`.
- [x] 19. `src/infra/adapters/reserva-client/set-proof/set-reserva-proof.adapter.ts`.
- [x] 20. `src/infra/adapters/validation-queue/publish/publish-validation-event.adapter.ts`.

### Fase 5 — `injection` — applications + wiring
- [x] 21. `src/applications/middlewares/internal-api-key.middleware.ts`.
- [x] 22. `src/applications/middlewares/static-file-validation.middleware.ts` (DP4).
- [x] 23. `src/applications/dto/inject-comprovante.dto.ts` (DP3).
- [x] 24. `src/applications/controllers/comprovante/upload/upload-comprovante.controller.ts` (202).
- [x] 25. `src/applications/routes/comprovante.route.ts` + `src/applications/routes/routes.ts`.
- [x] 26. `src/config/container.ts` (wiring).

### Fase 6 — `pagamentos` — remoção
- [x] 27. Remover `submit-manual-payment.usecase.ts` (+spec), `file-storage.port.ts`, `google-drive-file-storage.adapter.ts` (+spec), `manual-payment-repository.port.ts`, `mongo-manual-payment-repository.adapter.ts` (+spec), schema de manual payment, `create-manual-payment.dto.ts` (+spec).
- [x] 28. `src/domain/models/manual-payment.model.ts` — reduzir para `IManualPaymentView` + `IReviewManualPaymentInput`.
- [x] 29. `src/config/env.ts` — remover `googleDrive.*`; `+ injectionApiUrl`, `injectionInternalApiKey`, `minioPublicUrl`, `minioBucket`.

### Fase 7 — `pagamentos` — proxy + revisão via `agendamentos`
- [x] 30. `src/domain/ports/output/injection-client.port.ts` + `src/infra/adapters/injection/forward-comprovante.adapter.ts`.
- [x] 31. `src/domain/ports/output/reserve.port.ts` — `+ IListReservesForReviewPort`; `src/infra/adapters/reserve/list-reserves-for-review.adapter.ts`.
- [x] 32. `src/domain/usecases/manual-payment/query-manual-payments.usecase.ts` — reescrever (fonte `agendamentos`).
- [x] 33. `src/domain/usecases/manual-payment/review-manual-payment.usecase.ts` — reescrever (sem `manual_payments`).
- [x] 34. `src/applications/controllers/transaction-history/transaction-history.controller.ts` — `create`→proxy, `proof`→302, `list`/`read`/`review` ajustados.
- [x] 35. `src/config/container.ts` — remover/adicionar wiring.
- [x] 36. `src/applications/routes/routes.spec.ts` e demais specs de rota/controller — ajustar (feito no `/speckit-unit-tests`).

### Fase 8 — infra
- [x] 37. `beach-center-server/docker-compose.dev.yml` — `minio`, `minio-init`, `rabbitmq`, `injection`; envs de `pagamentos`; volumes.
- [x] 38. `beach-center-server/.env.dev.example` — novas envs.

### Fase 9 — conformidade
- [x] 39. `npm run lint` limpo em `injection`, `pagamentos`, `agendamentos`.
- [x] 40. `npx tsc --noEmit` (ou `npm run build`) sem erro nos 3 serviços.
- [x] 41. `git status` — só os arquivos previstos; `beach-center-app` intacto (verificado).

---

## Desvios / notas de implementação (`/speckit-implement` — 2026-09-10)

- **Porta do `injection` = 5006** (não 5005): o `docker-compose.dev.yml` já declara `5005` no
  serviço `campeonatos` (profile-gated). Ajustado em `.env.example`, `env.ts`, `main.ts` e no
  compose.
- **Item 36 (ajuste de specs do `pagamentos`) NÃO feito** — é território do `/speckit-unit-tests`
  (o `/speckit-implement` não escreve testes). Specs que precisam de rework:
  `src/config/container.spec.ts` (chave `submitManualPayment` → `forwardComprovante`),
  `src/applications/controllers/transaction-history/transaction-history.controller.spec.ts`
  (novo comportamento proxy/302), `src/applications/routes/routes.spec.ts` (revisar). Specs de
  módulos removidos já foram deletados. `src/test/fixtures.ts` foi atualizado (era bloqueador de
  `tsc`): `makeManualPayment` → `makeManualPaymentView`.
- **`check-public-link-authorization.adapter` + `ICheckPublicLinkAuthorizationPort` removidos do
  `pagamentos`** — ficaram órfãos (só o `SubmitManualPaymentUsecase` usava). O `injection` tem o
  seu próprio `check-public-link` adapter.
- **`IManualPaymentView` não carrega `slots`** — a reserva no `agendamentos` expõe só
  `scheduling_id[]`, não os slots expandidos. O painel de revisão perde o detalhe de horários por
  ora (enriquecer = task futura). Demais campos (`status`, `user`, `amount`, `proof_url`,
  `admin_note`) preservados.
- **Helpers de infra** adicionados além dos "adapters por verbo": `infra/adapters/object-storage/
  s3-client.ts` (factory singleton do `S3Client`) e `infra/adapters/reserva-client/
  agendamentos-http.ts` (instância `axios` com `x-api-key`). São plumbing de conexão, não regra —
  compartilhados pelos adapters de verbo.
- **6º adapter de `reserva-client`**: `revert-to-pending` (compensação DP10) — implícito no plano,
  explicitado como adapter próprio.
- **`injection` — repo Git próprio** já existe (`.git`), com `main` a criar no `/speckit-complete`
  (+ bump do gitlink no root). `node_modules` instalado (521 pacotes).
- **Gate local**: `tsc --noEmit` limpo nos 3 serviços; `npm run lint` limpo nos 3.
- **Infra não validada com Docker** (sem Docker no ambiente) — `docker-compose.dev.yml` validado
  só como YAML (`js-yaml`). Subida real de `minio`/`rabbitmq`/`injection` = validação manual no
  `/speckit-test`.

## Testes unitários (`/speckit-unit-tests` — 2026-09-10)

- **`beach-center-bff-injection`** — 21 suítes / 88 testes. Cobertura **97.74% stmt / 82.92%
  branch / 93.33% funcs / 98.28% lines** (≥ 80 global). Portas de infra 100% mockadas; usecase
  cobre AC-3/4/5/7/8 (upload, ordem status→proof→publish, autorização por token/e-mail,
  reconciliação, compensação nos 3 pontos de falha). Middlewares (AC-2: 400 por formato/tamanho,
  401 sem key), controller (202), dto, env, adapters (S3, RabbitMQ, `agendamentos` HTTP).
- **`beach-center-bff-pagamentos`** — 32 suítes / 137 testes. Cobertura **98.93 / 86.46 / 98.86 /
  99.06**. Novos/reescritos: `forward-comprovante.usecase` + `.adapter` (proxy), `query-manual-
  payments.usecase` (fonte `agendamentos` — AC-9), `review-manual-payment.usecase` (sem
  `manual_payments`), `list-reserves-for-review.adapter`, `transaction-history.controller`
  (proxy + 302), `container.spec` (chave `forwardComprovante`).
- **`beach-center-bff-agendamentos`** — 293 suítes / 1399 testes (+3). Cobertura **98.67 / 90.68 /
  98.94 / 98.73**. Novo `reserva.schema.spec.ts` (`toDomainReserve` + `proof_key`/`review_note`) +
  caso no `update-payment-metadata.controller.spec` (AC-4).
- **Ajuste no código durante os testes**: `proof_key` passou a ser `<uuid>.<ext>` (sem o prefixo
  `comprovantes/` que o DP8 sugeria) — o bucket já é `comprovantes`, então a URL fica
  `${MINIO_PUBLIC_URL}/comprovantes/<uuid>.<ext>` sem duplicar o segmento. Ajustado no usecase do
  `injection` e coerente com o `buildProofUrl` do `pagamentos`.
## Validação (`/speckit-validate` — 2026-09-10, modo autônomo "sem aprovação")

Revisão arquivo por arquivo (código de produção primeiro, por camada; testes depois) nos 4
repositórios afetados (`beach-center-bff-injection` 59 arquivos, `beach-center-bff-pagamentos`
33, `beach-center-bff-agendamentos` 6, `beach-center-server` 2 — todos `git add`-áveis, sem
sobra fora do previsto no plano).

**Conferência:** Constitution Check do plano revalidado (Princípios I/II/VI sem violação);
aderência às 15 decisões de planejamento (DP1–DP15); cobertura dos 12 ACs; isolamento hexagonal
(`no-restricted-imports` do ESLint não acusou nada); ordem de compensação no usecase
(status→proof→publish, com rollback nos 3 pontos de falha); proxy do `pagamentos` sem lógica de
arquivo; painel de revisão sem `manual_payments`; `docker-compose.dev.yml` válido como YAML,
serviço `injection` não exposto ao público pelo nginx.

**Correções aplicadas (3, autônomas):**

| # | Arquivo | Correção |
|---|---|---|
| 1 | `beach-center-bff-injection/src/domain/usecases/comprovante/inject-comprovante/inject-comprovante.usecase.ts` | **Bug real, achado pelo teste do caminho feliz**: `buildProofKey` gerava `comprovantes/<uuid>.<ext>`, e como o bucket já se chama `comprovantes`, a URL montada ficava com o segmento duplicado (`.../comprovantes/comprovantes/<uuid>.png`). Corrigido para `proof_key = <uuid>.<ext>` (só o object key; o bucket entra uma vez, via `buildFileUrl`). |
| 2 | `beach-center-bff-injection/src/applications/dto/inject-comprovante.dto.ts` | Troca do cast duplo `ALLOWED_PROOF_MIME_TYPES as unknown as string[]` pelo spread `[...ALLOWED_PROOF_MIME_TYPES]` — mais limpo, mesmo efeito. |
| 3 | `beach-center-bff-pagamentos/src/domain/usecases/manual-payment/query-manual-payments.usecase.ts` | Renomeado `DURATION_BY_STATUS_SOURCE` → `durationFromReserve` (o nome antigo sugeria relação com `status`, mas a função só lê `scheduling_id.length`). |

Após as correções: suítes reexecutadas (injection 88/88, pagamentos 137/137, agendamentos
1399/1399 — todas verdes, cobertura ≥ 80% mantida) e `npm run lint` limpo nos 3 serviços TS.

**Riscos observados, não corrigidos (fora do escopo de código):**
- Healthcheck do `minio` (`mc ready local`) segue o padrão oficial do MinIO, mas não foi testado
  com Docker real neste ambiente — se falhar, `minio-init`/`injection` não sobem; verificar no
  `/speckit-test`.
- `IManualPaymentView` sem `slots` (reserva só expõe `scheduling_id`) — reconhecido como redução
  desde o `/speckit-implement`, task futura para enriquecer.

**Estado:** todas as alterações **staged** (`git add`) nos 4 repositórios, **sem commit**.

## Testes de componente (`/speckit-component-tests` — 2026-09-10)

**N/A.** Task 100% backend (3 microsserviços). O DoD-6 proíbe alterar o `beach-center-app`, que
além disso **não tem** suíte Cypress/Cucumber (mesma situação das tasks 004–006b). Nenhum arquivo
`.feature`/step criado; nenhum atributo `data-cy` adicionado. A verificação de ponta a ponta
(front → `pagamentos` proxy → `injection` → MinIO/fila) fica no roteiro exploratório manual do
`/speckit-test`.

---

## Critérios de Aceite (formais)

### AC-1 — `pagamentos` livre de I/O de arquivo *(DoD-1)*
- **Given** o `beach-center-bff-pagamentos` após a task
- **When** se busca por `multer`, `busboy`, `googleapis`, upload/download de arquivo, `IFileStoragePort`, `google-drive`, `base64` de comprovante, e pela coleção `manual_payments`
- **Then** não há nenhuma dessas lógicas no repositório (só um **proxy** que repassa a requisição ao `injection` sem tocar no conteúdo do arquivo)
- **And** `submit-manual-payment.usecase.ts`, `file-storage.port.ts`, `google-drive-file-storage.adapter.ts` e `manual-payment-repository.port.ts` não existem mais.

### AC-2 — rota de ingestão protegida por validação estática *(DoD-2)*
- **Given** o `injection` no ar com `INJECTION_INTERNAL_API_KEY` configurada
- **When** um `POST /api/v1/agendamentos/comprovantes` chega com `x-api-key` válida e um `proof_file` com `mime_type` fora de `image/jpeg`/`image/png`/`application/pdf`
- **Then** a resposta é **`400`** com mensagem clara e **nenhum** upload/DB/fila acontece
- **And** o mesmo vale para um `proof_file.base64` cujo conteúdo decodificado passa de **5 MiB**
- **And** sem `x-api-key` (ou errada) a resposta é **`401`**.

### AC-3 — upload no MinIO *(DoD-3)*
- **Given** um `POST` válido (arquivo ok, reserva `pending`, autorização ok, dados conferem)
- **When** o usecase roda
- **Then** um objeto é gravado no bucket `MINIO_BUCKET` com key `comprovantes/<uuid>.<ext>` e `ContentType` igual ao `mime_type`
- **And** a `<ext>` corresponde ao mime (`jpg`/`png`/`pdf`).

### AC-4 — reserva marcada "Em Análise" + referência do arquivo *(DoD-4)*
- **Given** o upload concluído
- **When** o usecase segue
- **Then** o `injection` chama `PATCH /reservas/:id/status` `{status:"waiting_approve"}` **e** `PATCH /reservas/:id/payment` `{proof_key}` no `agendamentos` (via `x-api-key`)
- **And** a reserva fica com `status = waiting_approve`, ganha `number` (protocolo) e `proof_key`
- **And** o `injection` **não** abre conexão direta com o banco de reservas.

### AC-5 — evento publicado na fila após o DB *(DoD-5)*
- **Given** a reserva já marcada `waiting_approve` com `proof_key`
- **When** o usecase finaliza
- **Then** uma mensagem é publicada na fila **`comprovante.validar`** (durável, `persistent`) **depois** das chamadas do AC-4
- **And** o payload contém no mínimo `agendamento_id`, `usuario_id`, `file_url` e `timestamp` (ISO), além de `email`, `proof_key`, `bucket`, `mime_type`
- **And** a resposta HTTP ao chamador é **`202`** com corpo de "em processamento" — retornada sem aguardar qualquer processamento de IA.

### AC-6 — front-end intacto *(DoD-6)*
- **Given** o repositório `beach-center-app`
- **When** se roda `git status` / `git diff` após a task
- **Then** **nenhum** arquivo do `beach-center-app` foi alterado
- **And** o contrato de corpo que o front envia hoje (`POST /transaction-history` / `/public/transaction-history` com `proof_file.base64`) continua aceito (agora via proxy → `injection`).

### AC-7 — reserva `pending` obrigatória / autorização
- **Given** um `POST` cujo `reserve_id` aponta para uma reserva **inexistente**
- **Then** a resposta é **`404`** e nada é persistido/enfileirado
- **And** se a reserva existe mas `status !== "pending"` → **`409`**
- **And** se não há `public_reserve_token` válido **nem** `requester_email == reserve.email` → **`403`**
- **And** se `amount`/`duration_hours`/`slots` não conferem com a reserva → **`400`**.

### AC-8 — compensação em falha
- **Given** o upload no MinIO concluído
- **When** o `PATCH` no `agendamentos` falha (5xx/timeout)
- **Then** o `injection` tenta remover o objeto do MinIO (best-effort) e responde **`502`**
- **And** quando o `PATCH` passa mas o `publish` na fila falha → o `injection` reverte a reserva para `pending` (mantendo `number`), tenta remover o objeto e responde **`502`**.

### AC-9 — painel de revisão do admin funciona sem `manual_payments`
- **Given** reservas em `waiting_approve` com `proof_key`
- **When** o admin chama `GET /transaction-history` no `pagamentos`
- **Then** a lista vem do `agendamentos` (`GET /reservas?status=waiting_approve`), mapeada para a view de comprovante
- **And** `GET /transaction-history/:id/proof` responde **`302`** para `${MINIO_PUBLIC_URL}/${MINIO_BUCKET}/${proof_key}` (**`404`** se a reserva não tem `proof_key`)
- **And** `PATCH /transaction-history/:id/review` com `approved` põe a reserva em `approved` + `payment_method:"PIX"`; com `rejected` põe em `rejected`; `admin_note` é salvo em `reserva.review_note`
- **And** a coleção `manual_payments` não é lida nem escrita.

### AC-10 — Constituição / Hexagonal / ESLint
- **Given** os 3 serviços TS após a task
- **When** se roda `npm run lint` e `npm run build` (ou `tsc --noEmit`) em cada um
- **Then** ambos passam sem erro
- **And** no `injection`, `src/domain/**` não importa de `infra`/`applications` (regra ESLint), e há **um adapter por verbo** (`put-object`, `delete-object`, `read-reserva`, `check-public-link`, `set-in-review`, `set-proof`, `publish`).

### AC-11 — infra reproduzível
- **Given** `beach-center-server/docker-compose.dev.yml` após a task
- **When** se inspeciona o arquivo
- **Then** há serviços `minio`, `minio-init` (cria o bucket `comprovantes`), `rabbitmq` e `injection`
- **And** o `injection` tem bind-mount do código + volume de `node_modules` + `command` de `nodemon --legacy-watch` (hot-reload — Princípio IV)
- **And** o `injection` **não** é exposto ao público pelo gateway (só o `pagamentos` o consome).

### AC-12 — cobertura de testes ≥ 80% no `injection`
- **Given** a suíte do `beach-center-bff-injection` (gerada em `/speckit-unit-tests`)
- **When** se roda `npm run test:coverage`
- **Then** `statements`/`branches`/`functions`/`lines` ≥ 80% global
- **And** cada `AC-*` acima tem ao menos um teste correspondente (usecase com portas mockadas; middleware de validação; controller; adapters).

---

## Rastreabilidade DoD → AC

| DoD (US02) | AC |
|---|---|
| pagamentos livre de manipulação de arquivo físico | AC-1 |
| injection: rota de upload + validação (formato + 5 MB) | AC-2, AC-7 |
| arquivo salvo no MinIO | AC-3 |
| DB marcado "Pendente" + referência do arquivo | AC-4 |
| mensagem publicada na fila após o DB | AC-5 |
| front-end intacto | AC-6 |
| (consequências C3/C5 + qualidade) | AC-8, AC-9, AC-10, AC-11, AC-12 |

---

## Próximo passo

`/speckit-implement` — Fases 1→9. Depois `/speckit-unit-tests` (`injection` + `pagamentos` +
`agendamentos`), `/speckit-component-tests` (**N/A**), `/speckit-validate`, `/speckit-test`,
`/speckit-complete`, `/speckit-documentation`.
