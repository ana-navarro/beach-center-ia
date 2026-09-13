# Comprovante de Pagamento Manual

## Visão geral

Conjunto de endpoints do serviço `services/beach-center-bff-pagamentos` (base `/api/v1`) para
**submissão** e **revisão** de comprovantes de pagamento manual (ex.: PIX fora do gateway,
comprovante de depósito) anexados a uma reserva.

**Nota de nomenclatura importante:** apesar do prefixo de rota `transaction-history`, estes
endpoints **não** expõem o log de auditoria interno de transações (`ITransactionHistory` /
coleção `payment_transaction_history`, que registra os eventos de checkout/webhook e não é
exposto por nenhuma rota pública hoje). Eles operam sobre **comprovantes de pagamento manual**,
que — desde a task 008 — são apenas uma **visão derivada da reserva** em
`beach-center-bff-agendamentos` (não existe mais coleção própria). Inconsistência de nomenclatura
pré-existente no código-fonte, preservada aqui só para documentar o comportamento real da API.

> **Desacoplamento do upload (task 008 — US02):** o recebimento, a validação estática e o
> armazenamento do arquivo do comprovante **saíram deste serviço**. `beach-center-bff-pagamentos`
> agora só tem a **decisão** de aprovar/rejeitar (status lógico do pagamento) — o I/O de arquivo é
> do novo `beach-center-bff-injection` (ver [`../beach-center-bff-injection/comprovante.md`](../beach-center-bff-injection/comprovante.md)).
> A coleção `manual_payments`/`MongoManualPaymentRepositoryAdapter` e o storage no Google Drive
> (`GoogleDriveFileStorageAdapter`) foram **removidos**. O front-end **não muda**: as mesmas duas
> rotas de submissão continuam existindo aqui, mas agora são um **proxy fino** para o `injection`.

## Autenticação/autorização

- `POST /api/v1/transaction-history`: `authMiddleware` (Bearer) — autentica o usuário e repassa
  `usuario_id`/`requester_email` no proxy; **não** valida mais se é o dono da reserva (isso agora
  é responsabilidade do `injection`).
- `POST /api/v1/public/transaction-history`: **pública**, sem `authMiddleware`. Mesmo proxy, sem
  contexto de usuário (o corpo precisa trazer `public_reserve_token`; a autorização é validada no
  `injection`).
- `GET /api/v1/transaction-history`, `GET /api/v1/transaction-history/:id`,
  `GET /api/v1/transaction-history/:id/proof`, `PATCH /api/v1/transaction-history/:id/review`:
  `authMiddleware` + `requireRole("ADMIN")` — apenas usuários com `user_type: "ADMIN"`.

## Endpoints

### POST /api/v1/transaction-history

**Proxy fino (task 008).** Repassa a requisição para
`POST {INJECTION_API_URL}/agendamentos/comprovantes` no `beach-center-bff-injection`, com
`x-api-key: INJECTION_INTERNAL_API_KEY`. Aciona `ForwardComprovanteUsecase` →
`ForwardComprovanteAdapter`.

O controller (`TransactionHistoryController.create`) monta o contexto a partir da requisição
autenticada e o repassa junto ao corpo:
- `usuario_id` = `req.authUser.id_firestore` (uid do Firebase), quando autenticado.
- `requester_email` = `req.authUser.email`, quando autenticado.

**Nenhuma lógica de arquivo roda aqui** — o corpo (incluindo `proof_file.base64`) segue intacto
para o `injection`, que faz toda a validação/upload/gravação/enfileiramento. A resposta HTTP
(status e corpo) é a **mesma** que o `injection` devolveu — ver
[`../beach-center-bff-injection/comprovante.md`](../beach-center-bff-injection/comprovante.md)
para o contrato completo (corpo da requisição, regras de negócio, erros).

**Parâmetros de path/query** — nenhum.

**Corpo da requisição** — repassado como está para o `injection` (ver o documento dele); em
resumo: `reserve_id`, `reserve_number`, `amount`, `duration_hours`, `user{name,email,phone}`,
`slots[]`, `proof_file{file_name,mime_type,base64}`, `public_reserve_token?` (rota pública).

**Resposta de sucesso** — repassada do `injection`: `202 Accepted`
```json
{
  "message": "Comprovante recebido e em processamento",
  "data": {
    "agendamento_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "numero_protocolo": "0348871290",
    "proof_key": "3f2c1a9e-....png",
    "file_url": "http://localhost:9000/comprovantes/3f2c1a9e-....png",
    "queued": true
  }
}
```

**Erros possíveis** — repassados do `injection` (status HTTP e corpo idênticos):

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido, formato/tamanho do comprovante inválido, ou dados divergentes da reserva |
| 401 / 403 | Falha de autenticação no `pagamentos` (Bearer inválido na rota privada); ou `403` do `injection` se nem e-mail nem `public_reserve_token` autorizam o acesso à reserva |
| 404 | Reserva não encontrada |
| 409 | Reserva não está `pending` |
| 500 | Erro inesperado no proxy (ex.: `injection` fora do ar) |
| 502 | `injection` recebeu mas falhou ao processar (upload/gravação/fila) — repassado |

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5001/api/v1/transaction-history" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve_number": "R-0001",
    "amount": 80,
    "duration_hours": 1,
    "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
    "slots": [{ "id": "665f1a2b3c4d5e6f7a8b9c00", "date": "2026-09-10", "start_time": "18:00", "end_time": "19:00", "court": "Quadra 1", "unit": "Unidade Centro" }],
    "proof_file": { "file_name": "comprovante.png", "mime_type": "image/png", "base64": "<base64>" }
  }'
```

---

### POST /api/v1/public/transaction-history

Mesmo controller/usecase do endpoint acima (`TransactionHistoryController.create` →
`ForwardComprovanteUsecase`), mas sem `authMiddleware` — não há `usuario_id`/`requester_email` no
contexto repassado. A autorização é feita pelo `injection` via `public_reserve_token` (obrigatório
no corpo neste fluxo).

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5001/api/v1/public/transaction-history" \
  -H "Content-Type: application/json" \
  -d '{
    "public_reserve_token": "abcdefghij0123456789abcdefghij01",
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve_number": "R-0001",
    "amount": 80,
    "duration_hours": 1,
    "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
    "slots": [{ "id": "665f1a2b3c4d5e6f7a8b9c00", "date": "2026-09-10", "start_time": "18:00", "end_time": "19:00", "court": "Quadra 1", "unit": "Unidade Centro" }],
    "proof_file": { "file_name": "comprovante.png", "mime_type": "image/png", "base64": "<base64>" }
  }'
```

---

### GET /api/v1/transaction-history

Lista comprovantes para o painel de revisão do admin (ADMIN). **Task 008:** não lê mais uma
coleção própria — aciona `ListManualPaymentsUsecase`, que chama
`GET {AGENDAMENTOS_API_URL}/reservas?status=<status>[&name=<search>]` (padrão `status=waiting_approve`
— o "pendente de revisão" real) e mapeia cada reserva para a **view de comprovante**
(`toManualPaymentView`).

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Não | `pending` \| `waiting_approve` \| `approved` \| `rejected`. Default: `waiting_approve` |
| `search` | string | Não | Repassado como `name` na busca de reservas em `agendamentos` |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovantes listados com sucesso",
  "data": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
      "reserve_number": "0348871290",
      "amount": 80,
      "duration_hours": 1,
      "status": "waiting_approve",
      "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
      "proof_url": "http://localhost:9000/comprovantes/3f2c1a9e-....png"
    }
  ]
}
```
> `id` e `reserve_id` são o **id da reserva** (não existe mais um id de comprovante separado).
> `proof_url` só aparece quando a reserva tem `proof_key`. `admin_note` (quando presente) vem do
> campo `review_note` da reserva.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/transaction-history?status=waiting_approve" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### GET /api/v1/transaction-history/:id

Retorna a view de comprovante de **uma reserva** por id (ADMIN). Aciona
`ReadManualPaymentUsecase`, que lê `GET {AGENDAMENTOS_API_URL}/reservas/:id`.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) **da reserva** em `agendamentos` |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovante encontrado com sucesso",
  "data": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "waiting_approve", "...": "..." }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` — reserva inexistente |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c0d" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### GET /api/v1/transaction-history/:id/proof

Redireciona para o arquivo do comprovante (ADMIN). **Task 008:** não serve mais bytes do Mongo
nem redireciona para o Google Drive — aciona `GetManualPaymentProofUsecase`, que monta a URL a
partir de `reserve.proof_key` (`${MINIO_PUBLIC_URL}/${MINIO_BUCKET}/${proof_key}`) e responde
`302`.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) da reserva |

**Resposta de sucesso** — `302 Found`, `Location` = URL do objeto no MinIO.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` (reserva inexistente) ou `"Arquivo do comprovante nao encontrado"` (reserva existe mas não tem `proof_key`) |

**Exemplo de chamada**
```bash
curl -i "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c0d/proof" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### PATCH /api/v1/transaction-history/:id/review

Aprova ou rejeita o comprovante de uma reserva (ADMIN). Aciona `ReviewManualPaymentUsecase`.

Regras de negócio:
1. `status` do corpo deve ser exatamente `"approved"` ou `"rejected"`; qualquer outro valor gera
   `400` (`"Status de revisao invalido"`).
2. A reserva (`id`) deve existir — `404` senão (`"Comprovante nao encontrado"`).
3. Se a reserva **não** estiver `pending`/`waiting_approve` e o status atual dela **divergir** do
   `status` da revisão, `409` (`"Este comprovante ja foi revisado"`) — idempotência/proteção
   contra revisão dupla.
4. Se a reserva está `pending` ou `waiting_approve` (o normal é `waiting_approve`): quando
   `approved`, grava `payment_method: "PIX"` (+ `review_note`, se enviado) via
   `PATCH .../reservas/:id/payment`, depois `PATCH .../reservas/:id/status` com o novo status.
   Quando `rejected`, só o `review_note` (se enviado) + a transição de status.
5. **Task 008:** não escreve mais em `manual_payments` (coleção descontinuada) — `admin_note` é
   persistido em `reserve.review_note`.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Sim | `"approved"` ou `"rejected"` |
| `admin_note` | string | Não | Observação do administrador — gravada em `reserve.review_note` |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovante revisado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "status": "approved",
    "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
    "proof_url": "http://localhost:9000/comprovantes/3f2c1a9e-....png"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"Status de revisao invalido"` |
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` — reserva inexistente |
| 409 | `"Este comprovante ja foi revisado"` |

**Exemplo de chamada**
```bash
curl -X PATCH "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c0d/review" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

## Referências

- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/transaction-history/transaction-history.controller.ts`
- Usecases: `.../domain/usecases/manual-payment/forward-comprovante.usecase.ts` (proxy — task 008), `.../query-manual-payments.usecase.ts` (`ListManualPaymentsUsecase`, `ReadManualPaymentUsecase`, `GetManualPaymentProofUsecase` — fonte: `agendamentos`), `.../review-manual-payment.usecase.ts`
- Client do injection: `.../domain/ports/output/injection-client.port.ts`, `.../infra/adapters/injection/forward-comprovante.adapter.ts`
- Client do agendamentos (revisão): `.../domain/ports/output/reserve.port.ts` (`IListReservesForReviewPort`), `.../infra/adapters/reserve/list-reserves-for-review.adapter.ts`, `.../infra/adapters/reserve/read-reserve.adapter.ts`
- Modelo: `services/beach-center-bff-pagamentos/src/domain/models/manual-payment.model.ts` (`IManualPaymentView` — view derivada da reserva, task 008)
- Middleware de auth: `services/beach-center-bff-pagamentos/src/applications/middlewares/auth.middleware.ts` (`authMiddleware`, `requireRole`)
- Env (task 008): `INJECTION_API_URL`, `INJECTION_INTERNAL_API_KEY`, `MINIO_PUBLIC_URL`, `MINIO_BUCKET` (`src/config/env.ts`)
- Serviço de upload (task 008): [`../beach-center-bff-injection/comprovante.md`](../beach-center-bff-injection/comprovante.md)
- Reserva (dono do dado, task 008): [`../beach-center-bff-agendamentos/reserva.md`](../beach-center-bff-agendamentos/reserva.md) (`proof_key`, `review_note` em `PATCH /reservas/:id/payment`)

**Removido nesta task:** `IFileStoragePort`/`GoogleDriveFileStorageAdapter`,
`IManualPaymentRepositoryPort`/`MongoManualPaymentRepositoryAdapter`,
`SubmitManualPaymentUsecase`, `create-manual-payment.dto.ts`, schema `manual-payment.schema.ts`
(coleção `manual_payments`), env `googleDrive.*`.
