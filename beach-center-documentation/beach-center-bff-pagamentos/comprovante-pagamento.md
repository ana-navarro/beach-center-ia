# Comprovante de Pagamento Manual

## Visão geral
Conjunto de endpoints do serviço `services/beach-center-bff-pagamentos` (base `/api/v1`) para
submissão e revisão de **comprovantes de pagamento manual** (ex.: PIX fora do gateway, comprovante
de depósito) anexados a uma reserva.

**Nota de nomenclatura importante:** apesar do prefixo de rota `transaction-history`, estes
endpoints **não** expõem o log de auditoria interno de transações (`ITransactionHistory` /
coleção `payment_transaction_history`, que registra os eventos de checkout/refund/webhook e não é
exposto por nenhuma rota pública hoje). Eles operam exclusivamente sobre
**comprovantes de pagamento manual** — modelo `IManualPayment`, coleção `manual_payment_validations`
— submetidos por clientes e revisados por administradores. Essa é uma inconsistência de nomenclatura
pré-existente no código-fonte, preservada aqui apenas para documentar o comportamento real da API.

## Autenticação/autorização
- `POST /api/v1/transaction-history`: `authMiddleware` (Bearer) — usuário autenticado deve ser o
  **dono da reserva** (e-mail do token bate com o e-mail da reserva) para ser autorizado.
- `POST /api/v1/public/transaction-history`: **pública**, sem `authMiddleware`. A autorização é
  feita via `public_reserve_token` (token do link público de reserva), validado contra
  `agendamentos` (`checkPublicLinkAuthorizationPort`).
- `GET /api/v1/transaction-history`, `GET /api/v1/transaction-history/:id`,
  `GET /api/v1/transaction-history/:id/proof`, `PATCH /api/v1/transaction-history/:id/review`:
  `authMiddleware` + `requireRole("ADMIN")` — apenas usuários com `user_type: "ADMIN"`.

## Endpoints

### POST /api/v1/transaction-history
Submete um comprovante de pagamento manual para uma reserva, autenticado via Bearer. Aciona
`SubmitManualPaymentUsecase`. O usuário autenticado deve ser o dono da reserva (e-mail do token
Firebase == e-mail da reserva).

### POST /api/v1/public/transaction-history
Mesmo controller/usecase do endpoint acima (`TransactionHistoryController.create`), mas sem
`authMiddleware`. A autorização é feita via `public_reserve_token` (obrigatório no corpo neste
fluxo, validado contra `GET {AGENDAMENTOS_API_URL}/links-reserva-publica/:token/reservas/:id/authorization`).

Regras de negócio aplicadas pelo usecase (`domain/usecases/manual-payment/submit-manual-payment.usecase.ts`),
comuns aos dois endpoints acima:
1. A reserva (`reserve_id`) deve existir em `agendamentos`, senão `404`.
2. A reserva deve estar `status: "pending"`, senão `409`.
3. Autorização: `public_reserve_token` válido para a reserva **ou** e-mail do usuário autenticado
   igual ao e-mail da reserva; caso contrário `403`.
4. Não pode já existir comprovante para a mesma reserva (`existsByReserveId`), senão `409`.
5. Conciliação de valores: `amount`, `duration_hours` e a lista de `slots` (por `id`) enviados
   devem corresponder exatamente aos dados da reserva (`reserve.total`, `reserve.scheduling_id`,
   mapeamento `duration_hours → amount` fixo `{1:80, 2:120, 3:160}`); qualquer divergência gera
   `400`.
6. Upload do arquivo do comprovante via `IFileStoragePort`
   (`google-drive-file-storage.adapter.ts`): se `GOOGLE_DRIVE_CLIENT_EMAIL`,
   `GOOGLE_DRIVE_PRIVATE_KEY` e `GOOGLE_DRIVE_FOLDER_ID` estiverem todos configurados, o arquivo é
   enviado ao Google Drive (`proof.storage: "google_drive"`, com `view_url`/`preview_url`);
   caso contrário, o arquivo cai em modo `storage: "database"`, salvando o `base64` diretamente
   no Mongo (`proof.data`).
7. **Compensação em caso de falha**: se o upload do comprovante ou a persistência no banco
   falharem, a reserva é automaticamente movida para `status: "rejected"`
   (`updateReserveStatusPort.execute(reserve, "rejected")`) antes de propagar o erro — evita que a
   reserva fique presa em estado intermediário. Esse erro não tratado propaga como `500` genérico.

**Parâmetros de path/query** — nenhum.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `public_reserve_token` | string (32 chars) | Somente na rota pública | Token do link público de reserva |
| `reserve_id` | string | Sim | Id da reserva em `agendamentos` |
| `reserve_number` | string | Sim | Número/código da reserva |
| `amount` | number | Sim | Um de `80`, `120`, `160` (deve bater com a reserva) |
| `duration_hours` | number | Sim | Um de `1`, `2`, `3` (deve bater com `scheduling_id.length` da reserva) |
| `user.name` | string | Sim | Nome do cliente |
| `user.email` | string (email) | Sim | E-mail do cliente |
| `user.phone` | string | Sim | Telefone do cliente |
| `slots` | array (1 a 3 itens) | Sim | Cada item: `id`, `date`, `start_time`, `end_time`, `court`, `unit` (todos string, obrigatórios) — `id` de cada slot deve bater com `scheduling_id` da reserva |
| `proof_file.file_name` | string | Sim | Nome do arquivo |
| `proof_file.mime_type` | string | Sim | Um de `image/jpeg`, `image/png`, `application/pdf` |
| `proof_file.base64` | string | Sim | Conteúdo em base64, no máximo ~7.000.000 caracteres (≈ 5 MB, mensagem `"O comprovante deve ter no maximo 5 MB"`) |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Comprovante enviado para validacao",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c10",
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve_number": "R-0001",
    "amount": 80,
    "duration_hours": 1,
    "status": "pending",
    "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
    "slots": [{ "id": "665f1a2b3c4d5e6f7a8b9c00", "date": "2026-09-10", "start_time": "18:00", "end_time": "19:00", "court": "Quadra 1", "unit": "Unidade Centro" }],
    "proof": { "file_name": "R-0001-comprovante.png", "mime_type": "image/png", "storage": "database" },
    "created_at": "2026-09-06T12:00:00.000Z",
    "updated_at": "2026-09-06T12:00:00.000Z"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido (yup); ou `"Os dados do pagamento nao correspondem a reserva"` (divergência de valores/slots) |
| 403 | `"Acesso negado para esta reserva"` — nem `public_reserve_token` nem e-mail do usuário autorizam |
| 404 | `"Reserva nao encontrada"` |
| 409 | `"A reserva nao esta pendente de pagamento"` (reserva já aprovada/rejeitada) ou `"Ja existe um comprovante para esta reserva"` |
| 500 | Falha no upload/persistência do comprovante (ex.: credenciais do Google Drive inválidas) — a reserva é automaticamente rejeitada como compensação |

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

Chamada equivalente à rota pública (sem Bearer, com `public_reserve_token`):
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
Lista comprovantes de pagamento manual (ADMIN). Aciona `ListManualPaymentsUsecase`, que consulta
`MongoManualPaymentRepositoryAdapter.list`, ordenado por `createdAt` decrescente e limitado a
**200 registros**.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Não | Filtra por `pending`, `approved` ou `rejected` |
| `search` | string | Não | Busca (regex case-insensitive) em `reserve_id`, `reserve_number`, `user.name`, `user.email`, `user.phone` |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovantes listados com sucesso",
  "data": [
    { "id": "665f1a2b3c4d5e6f7a8b9c10", "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "pending", "amount": 80, "duration_hours": 1, "...": "..." }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` (`"Acesso negado"`) |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/transaction-history?status=pending" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### GET /api/v1/transaction-history/:id
Retorna um comprovante específico por id (ADMIN). Aciona `ReadManualPaymentUsecase`.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) do comprovante |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovante encontrado com sucesso",
  "data": { "id": "665f1a2b3c4d5e6f7a8b9c10", "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "pending", "...": "..." }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c10" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### GET /api/v1/transaction-history/:id/proof
Faz o download/redirecionamento do arquivo do comprovante (ADMIN). Aciona
`GetManualPaymentProofUsecase`.

Comportamento por `proof.storage`:
- `"google_drive"` (e `view_url` presente): responde `302` com `Location` = `proof.view_url`
  (`res.redirect`).
- `"database"`: decodifica `proof.data` (base64) e devolve os bytes do arquivo com
  `Content-Type: <proof.mime_type>` (respeita o mime real do arquivo, ex.: `application/pdf`),
  `Content-Disposition: inline; filename*=UTF-8''<nome codificado>` e
  `Cache-Control: private, no-store`.
- Se não houver `proof.data` disponível (nem Drive nem base64 salvo), responde `404` com
  `{"message": "Arquivo do comprovante nao encontrado"}` — tratado diretamente no controller,
  fora do `handleHttpError` padrão.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) do comprovante |

**Resposta de sucesso** — `200 OK` (bytes do arquivo) ou `302` (redirect para o Google Drive)

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` (comprovante inexistente) ou `"Arquivo do comprovante nao encontrado"` (comprovante existe, mas sem dado de arquivo disponível) |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c10/proof" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  --output comprovante
```

---

### PATCH /api/v1/transaction-history/:id/review
Aprova ou rejeita um comprovante pendente (ADMIN). Aciona `ReviewManualPaymentUsecase`.

Regras de negócio:
1. `status` do corpo deve ser exatamente `"approved"` ou `"rejected"`; qualquer outro valor gera
   `400` (`"Status de revisao invalido"`).
2. O comprovante deve existir (`404` senão) e estar `status: "pending"` — se já revisado, `409`
   (`"Este comprovante ja foi revisado"`).
3. A reserva associada deve existir (`404` senão).
4. Se a reserva **não** estiver mais `pending` e o status atual dela **divergir** do `status` da
   revisão, retorna `409` (`"A reserva nao esta pendente de revisao"`) — proteção contra revisar um
   comprovante cuja reserva mudou de estado por outro fluxo nesse meio-tempo.
5. Se a reserva ainda está `pending`: quando aprovado, primeiro grava `payment_method: "PIX"` na
   reserva (`PATCH .../reservas/:id/payment`) e depois `PATCH .../reservas/:id/status` com o novo
   status (`approved` ou `rejected`).
6. Persiste a revisão (`status`, `admin_note`, `reviewed_at`) no comprovante.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | Id (ObjectId) do comprovante |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Sim | `"approved"` ou `"rejected"` |
| `admin_note` | string | Não | Observação do administrador (ex.: motivo da rejeição) |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovante revisado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c10",
    "status": "approved",
    "admin_note": null,
    "reviewed_at": "2026-09-06T13:00:00.000Z",
    "...": "..."
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"Status de revisao invalido"` |
| 401 | Sem `Authorization` ou token inválido/expirado |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | `"Comprovante nao encontrado"` ou `"Reserva nao encontrada"` |
| 409 | `"Este comprovante ja foi revisado"` ou `"A reserva nao esta pendente de revisao"` |

**Exemplo de chamada**
```bash
curl -X PATCH "http://localhost:5001/api/v1/transaction-history/665f1a2b3c4d5e6f7a8b9c10/review" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

## Referências
- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/transaction-history/transaction-history.controller.ts`
- DTO: `services/beach-center-bff-pagamentos/src/applications/dto/create-manual-payment.dto.ts`
- Usecases: `services/beach-center-bff-pagamentos/src/domain/usecases/manual-payment/submit-manual-payment.usecase.ts`, `.../query-manual-payments.usecase.ts`, `.../review-manual-payment.usecase.ts`
- Repositório: `services/beach-center-bff-pagamentos/src/infra/adapters/manual-payment/mongo-manual-payment-repository.adapter.ts`
- Armazenamento de arquivo: `services/beach-center-bff-pagamentos/src/infra/adapters/file-storage/google-drive-file-storage.adapter.ts`
- Modelos: `services/beach-center-bff-pagamentos/src/domain/models/manual-payment.model.ts`
- Middleware de auth: `services/beach-center-bff-pagamentos/src/applications/middlewares/auth.middleware.ts` (`authMiddleware`, `requireRole`)
