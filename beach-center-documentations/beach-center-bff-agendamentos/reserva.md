# Reserva

## Visão geral

O recurso `reserva` (`/api/v1/reservas`) representa a reserva de um ou mais horários (`agendamentos`) por um cliente, incluindo protocolo público, ciclo de vida de status (`pending → approved/rejected/cancelled`), integração com pagamento e estorno. É exposto pelo serviço **beach-center-bff-agendamentos**.

**Nota de integração cross-serviço:** três destes endpoints não são consumidos pelo frontend, e sim pelo **beach-center-bff-pagamentos**, autenticando-se com uma chave interna (`x-api-key`) em vez de um usuário Firebase:

- `PATCH /reservas/:id/payment` — `pagamentos` grava aqui os metadados do pagamento (`payment_id`, `checkout_id`, `payment_method`, `refund_id`, `refund_status`, `refunded_at`) após um checkout, webhook Getnet ou reembolso.
- `PATCH /reservas/:id/status` — o webhook do Getnet em `pagamentos` chama esta rota para aprovar/rejeitar a reserva, o que bloqueia/libera os agendamentos vinculados.
- `GET /reservas/:id` — `pagamentos` lê os dados da reserva durante o checkout, sem Bearer, só com `x-api-key`.

Além disso, ao cancelar/deletar uma reserva `approved` com pagamento, `agendamentos` chama de volta `POST {PAGAMENTOS_API_URL}/refunds` em `pagamentos` (ver `infra/adapters/payment/refund-reserve-payment.adapter.ts`) usando `PAGAMENTOS_INTERNAL_API_KEY`.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /reservas` | `authMiddleware` (Bearer — qualquer usuário autenticado) |
| `GET /reservas/protocol/:number` | pública |
| `PATCH /reservas/protocol/:number/cancel` | pública |
| `PATCH /reservas/:id/payment` | `internalApiKeyMiddleware` (somente `x-api-key`) |
| `PATCH /reservas/:id/delete` | `authMiddleware` + `requireRole('ADMIN')` |
| `GET /reservas/:id` | `adminOrInternalApiKey` (Bearer+ADMIN **ou** `x-api-key`) |
| `GET /reservas` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /reservas/:id` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /reservas/:id/status` | `adminOrInternalApiKey` (Bearer+ADMIN **ou** `x-api-key`) |

`internalApiKeyMiddleware` e `adminOrInternalApiKey` validam o header `x-api-key` contra `AGENDAMENTOS_INTERNAL_API_KEY`. Quando a autenticação falha nessas duas rotas puramente por API key (`internalApiKeyMiddleware`), a resposta é `401 {"message": "Unauthorized"}` (em inglês, escrita própria do middleware — **não** passa por `handle-http-error.ts`, que produz mensagens em PT-BR).

## Endpoints

### POST /api/v1/reservas

Cria uma reserva para 1 a 3 horários (`scheduling_id`). Aciona `CreateReserveUsecase`, que delega toda a validação a `ReserveSchedulingValidator.validateForReserve`: limite de 3 horários e duplicidade (redundante com a validação do DTO), existência dos agendamentos, data não passada, ausência de conflito com evento agendado (`EventConflictService`), e disponibilidade (`available === true`). Se aprovado, gera `number` (protocolo) via `nanoid` (10 dígitos numéricos) quando não informado, e persiste com `status: "pending"`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome do cliente |
| `email` | string | Sim | E-mail válido |
| `phone` | string | Sim | Telefone |
| `scheduling_id` | string[] | Sim | 1 a 3 ObjectIds (24 hex) de agendamentos, sem repetição |
| `total` | number | Sim | Valor total (`>= 0`) |
| `price` | number | Não | Preço unitário/base (`>= 0`) |
| `discount` | number \| null | Não | Desconto aplicado (`>= 0`) |
| `equipment` | object | Sim | `{ self_equipment: boolean, equipment?: string[] }` — se `self_equipment: false`, `equipment` (lista) é obrigatório e precisa ter ao menos 1 item |
| `number` | string | Não | Protocolo customizado; se omitido, gerado automaticamente |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Reserva criada com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "name": "João Silva",
    "email": "joao@example.com",
    "phone": "11999998888",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c01"],
    "price": 80,
    "discount": null,
    "total": 80,
    "status": "pending",
    "equipment": { "self_equipment": true },
    "number": "1234567890"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | DTO inválido (`Dados inválidos`) — campo obrigatório ausente, `scheduling_id` com mais de 3 itens ou repetido, `equipment` inconsistente |
| 400 | `"E permitido selecionar no maximo 3 horarios"` / `"Nao e permitido repetir horarios na mesma reserva"` (redundante com o DTO, mas também validado no usecase) |
| 400 | `"Nao e permitido criar reserva para datas que ja passaram"` |
| 404 | `{"message": "Agendamento não encontrado", "id": [<scheduling_id enviados>]}` — shape especial, **não** segue o padrão `{message}` de `handle-http-error.ts` — ocorre quando algum `scheduling_id` não existe |
| 409 | `"Um ou mais horarios selecionados possuem excecao de agendamento"` — algum horário coincide com evento agendado `CONFIRMED` |
| 409 | `"Um ou mais horarios selecionados nao estao disponiveis"` — algum horário já está `available: false` (reservado) |
| 401 | Sem token (`"Token nao fornecido"`) |

**Exemplo de chamada**

```bash
curl -X POST "https://<host>/api/v1/reservas" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "João Silva",
    "email": "joao@example.com",
    "phone": "11999998888",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c01"],
    "total": 80,
    "equipment": { "self_equipment": true }
  }'
```

---

### GET /api/v1/reservas/protocol/:number

Busca uma reserva pelo protocolo público (`number`). Aciona `FindReserveByProtocolUsecase`, que além dos dados da reserva calcula `can_cancel` e `cancellation_deadline` com base na regra de janela de cancelamento (2h antes do agendamento mais cedo vinculado; limite exato inclusive, `<=`). Reservas `cancelled` ou sem agendamentos vinculados sempre retornam `can_cancel: false`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `number` | string | Sim | Número do protocolo da reserva |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva encontrada com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "number": "1234567890",
    "status": "pending",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c01"],
    "can_cancel": true,
    "cancellation_deadline": "2026-09-08T16:00:00.000Z"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `number` ausente/vazio no path (`"Numero de protocolo e obrigatorio"`, shape `{message: "Dados invalidos", errors: [...]}`) |
| 404 | `{"message": "Reserva nao encontrada"}` — protocolo inexistente |

**Exemplo de chamada**

```bash
curl "https://<host>/api/v1/reservas/protocol/1234567890"
```

---

### PATCH /api/v1/reservas/protocol/:number/cancel

Cancela uma reserva pelo protocolo, sem autenticação (fluxo público, ex.: cliente cancelando pelo link recebido). Aciona `DeleteReserveUsecase.executeByProtocol`, que localiza a reserva pelo protocolo, bloqueia se já estiver `cancelled`, e delega a `execute(id)` — mesma lógica de `PATCH /reservas/:id/delete` (janela de 2h, estorno se aplicável).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `number` | string | Sim | Número do protocolo da reserva |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva cancelada com sucesso",
  "data": [
    { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "cancelled", "...": "..." }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `number` ausente/vazio (`"Numero de protocolo e obrigatorio"`) |
| 400 | `"Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"` |
| 404 | `{"message": "Reserva nao encontrada"}` — protocolo inexistente |
| 409 | `{"message": "Reserva ja foi cancelada"}` — reserva já estava `cancelled` |
| 409 | `"Nao foi possivel estornar: reserva aprovada sem identificador de pagamento"` — reserva `approved`, `total > 0`, sem `payment_id` e `payment_method !== "PIX"` |
| 502 | `"Nao foi possivel estornar o pagamento da reserva"` — o gateway (via `pagamentos`) retornou `refund_status: "failed"` |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/protocol/1234567890/cancel"
```

---

### PATCH /api/v1/reservas/:id/payment

**Consumido por `beach-center-bff-pagamentos`.** Atualiza os metadados de pagamento/estorno de uma reserva (chamado após checkout, webhook Getnet aprovado, ou reembolso processado). Aciona `UpdateReservePaymentMetadataUsecase`, que apenas valida o `id` e delega à persistência — não há regra de negócio adicional.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `payment_id` | string | Não | Identificador do pagamento no gateway |
| `checkout_id` | string | Não | Identificador do checkout |
| `payment_method` | string | Não | `"PIX"` \| `"CREDIT_CARD"` |
| `refund_id` | string | Não | Identificador do estorno |
| `refund_status` | string | Não | `"pending"` \| `"approved"` \| `"failed"` \| `"skipped"` |
| `refunded_at` | string (data) | Não | Data/hora do estorno |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Metadados de pagamento atualizados com sucesso",
  "data": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "payment_id": "mock_abc123", "...": "..." }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido (`payment_method`/`refund_status` fora do `oneOf`) |
| 401 | `{"message": "Unauthorized"}` — `x-api-key` ausente ou incorreta |
| 404 | `{"message": "Reserva nao encontrada"}` |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/665f1a2b3c4d5e6f7a8b9c0d/payment" \
  -H "x-api-key: <AGENDAMENTOS_INTERNAL_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"payment_id": "mock_abc123", "checkout_id": "chk_1", "payment_method": "PIX"}'
```

---

### PATCH /api/v1/reservas/:id/delete

Cancela (soft) uma reserva por ID, restrito a ADMIN. Aciona `DeleteReserveUsecase.execute`: valida que os agendamentos vinculados ainda existem, aplica a janela de cancelamento de 2h (a menos que chamado internamente com `skipCancellationTimeValidation`, usado pelo fluxo `PATCH /dias/close-date` com `cancel_reserves: true`), processa estorno se a reserva estiver `approved` com valor > 0, e libera os agendamentos elegíveis (`SchedulingAvailabilityService.releaseSchedulingsIfAllowed`).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva deletada com sucesso",
  "data": [
    { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "cancelled", "...": "..." }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"ID inválido"` — `:id` não é ObjectId de 24 hex |
| 400 | `"Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"` |
| 404 | `{"message": "Reserva não encontrada"}` |
| 404 | `"Agendamento da reserva nao encontrado"` — algum agendamento vinculado à reserva não existe mais |
| 409 | `"Nao foi possivel estornar: reserva aprovada sem identificador de pagamento"` |
| 502 | `"Nao foi possivel estornar o pagamento da reserva"` |
| 401 / 403 | Sem token / não-ADMIN |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/665f1a2b3c4d5e6f7a8b9c0d/delete" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### GET /api/v1/reservas/:id

Busca uma reserva por ID. **Consumido também por `beach-center-bff-pagamentos`** (com `x-api-key`, sem Bearer) durante o fluxo de checkout. Aciona `ReadReserveUsecase`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva encontrada com sucesso",
  "data": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "pending", "...": "..." }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"ID inválido"` |
| 401 | Sem Bearer nem `x-api-key` válidos (`"Token nao fornecido"`) |
| 403 | Bearer de não-ADMIN sem `x-api-key` |
| 404 | `{"message": "Reserva não encontrada"}` |

**Exemplo de chamada**

```bash
curl "https://<host>/api/v1/reservas/665f1a2b3c4d5e6f7a8b9c0d" \
  -H "x-api-key: <AGENDAMENTOS_INTERNAL_API_KEY>"
```

---

### GET /api/v1/reservas

Lista reservas com filtros opcionais, restrito a ADMIN. Aciona `ListReservesUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Não | `"pending"` \| `"approved"` \| `"rejected"` \| `"cancelled"` |
| `name` | string | Não | Filtro por nome |
| `email` | string | Não | Filtro por e-mail |
| `phone` | string | Não | Filtro por telefone |
| `scheduling_id` | string | Não | Filtro por horário vinculado |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reservas listadas com sucesso",
  "data": [{ "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "pending", "...": "..." }]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 / 403 | Sem token / não-ADMIN |

**Exemplo de chamada**

```bash
curl "https://<host>/api/v1/reservas?status=pending&name=jo" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### PATCH /api/v1/reservas/:id

Atualização parcial de uma reserva, restrito a ADMIN. Aciona `UpdateReserveUsecase`: faz merge dos campos enviados sobre a reserva atual, revalida limite de 3 horários, existência, data não passada e conflito com evento (a menos que o novo `status` seja `cancelled`/`rejected`), e — quando os `scheduling_id` mudam — valida disponibilidade **apenas dos horários adicionados** (não revalida os que já estavam na reserva). Libera os horários removidos e bloqueia os atuais.

> ⚠️ **Comportamento pré-existente (não corrigido nesta documentação):** o schema `update-reserva.dto.ts` declara `equipment` como `.optional()`, mas o yup ainda valida o shape interno (`self_equipment` como `.required()`) quando a chave `equipment` está **totalmente ausente** do corpo. Ou seja, um PATCH parcial que omite `equipment` falha com `400` (`"equipment.self_equipment is a required field"`) em vez de aplicar o merge parcial esperado — enviar `equipment` completo mesmo em updates parciais que não o alteram.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Não | — |
| `email` | string | Não | — |
| `phone` | string | Não | — |
| `scheduling_id` | string[] | Não | 1 a 3 ObjectIds, sem repetição |
| `price` | number | Não | `>= 0` |
| `discount` | number \| null | Não | `>= 0` |
| `status` | string | Não | `"pending"` \| `"approved"` \| `"rejected"` \| `"cancelled"` |
| `total` | number | Não | `>= 0` |
| `equipment` | object | Ver nota acima | `{ self_equipment, equipment? }` — na prática sempre exigido pelo quirk do DTO |
| `number` | string | Não | — |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva encontrada com sucesso",
  "data": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "name": "Nome Novo", "...": "..." },
  "list": [{ "id": "665f1a2b3c4d5e6f7a8b9c0d", "...": "..." }]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"ID invalido"` |
| 400 | `"equipment.self_equipment is a required field"` (ver nota acima) |
| 400 | `"E permitido selecionar no maximo 3 horarios"` / `"Nao e permitido repetir horarios na mesma reserva"` |
| 400 | `"Nao e permitido criar reserva para datas que ja passaram"` |
| 404 | `{"message": "Reserva nao encontrada"}` — reserva ou algum `scheduling_id` informado não existe |
| 409 | `"Um ou mais horarios selecionados possuem excecao de agendamento"` |
| 409 | `"Um ou mais horarios selecionados nao estao disponiveis"` — só para os horários **adicionados** |
| 401 / 403 | Sem token / não-ADMIN |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/665f1a2b3c4d5e6f7a8b9c0d" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Nome Novo", "equipment": {"self_equipment": true}}'
```

---

### PATCH /api/v1/reservas/:id/status

**Consumido por `beach-center-bff-pagamentos`** (webhook Getnet) para aprovar/rejeitar/cancelar uma reserva após o processamento do pagamento. Também acessível a ADMIN via Bearer. Aciona `UpdateReserveAndSchedulingStatusUsecase`: revalida os horários vinculados quando o novo status é `pending`/`approved` (existência, data não passada, sem conflito de evento, e — se a reserva vinha de `cancelled`/`rejected` — disponibilidade); bloqueia os agendamentos ao aprovar/deixar pendente, ou libera-os ao rejeitar/cancelar.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Sim | `"pending"` \| `"approved"` \| `"rejected"` \| `"cancelled"` |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Status atualizado com sucesso",
  "data": {
    "reserve": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "approved", "...": "..." },
    "scheduling": [{ "id": "665f1a2b3c4d5e6f7a8b9c01", "available": false, "...": "..." }]
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"ID inválido"` |
| 400 | Corpo inválido — mensagem crua do yup (`error.message`), este controller **não** usa o padrão `"Dados inválidos"` dos demais — ex.: `"status must be one of the following values: ..."` |
| 404 | `{"message": "Reserva não encontrada"}` |
| 404 | `"Agendamento da reserva nao encontrado"` |
| 409 | `"Um ou mais horarios selecionados possuem excecao de agendamento"` |
| 409 | `"Um ou mais horarios selecionados nao estao disponiveis"` — ao reverter de `cancelled`/`rejected` para `pending`/`approved` |
| 401 | `{"message": "Unauthorized"}` (se via `x-api-key` incorreta) ou `"Token nao fornecido"` (via Bearer) |
| 403 | Bearer de não-ADMIN |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/665f1a2b3c4d5e6f7a8b9c0d/status" \
  -H "x-api-key: <AGENDAMENTOS_INTERNAL_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

## Referências

- Rota: `src/applications/routes/reserva.route.ts`
- Controllers: `src/applications/controllers/reserva/create/create-reserve.controller.ts`, `.../find-by-protocol/find-by-protocol.controller.ts`, `.../cancel-by-protocol/cancel-reserve-by-protocol.controller.ts`, `.../payment-metadata/update-payment-metadata.controller.ts`, `.../delete/delete-reserve.controller.ts`, `.../read/read-reserve.controller.ts`, `.../list/list-reserves.controller.ts`, `.../update/update-reserve.controller.ts`, `src/applications/controllers/shared/update-reserve-and-scheduling-status.controller.ts`
- Usecases: `src/domain/usecases/reserva/create/create-reserve.usecase.ts`, `.../find-by-protocol/find-by-protocol.usecase.ts`, `.../delete/delete-reserve.usecase.ts`, `.../update-payment-metadata/update-payment-metadata.usecase.ts`, `.../read/read-reserve.usecase.ts`, `.../list/list-reserves.usecase.ts`, `.../update/update-reserve.usecase.ts`, `src/domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts`
- Regras compartilhadas: `src/domain/usecases/shared/reserve-scheduling.validator.ts`, `.../reserve-cancellation-window.ts`, `.../scheduling-availability.service.ts`, `.../event-conflict.service.ts`
- DTOs: `src/applications/dto/create-reserva.dto.ts`, `update-reserva.dto.ts`, `update-payment-metadata.dto.ts`, `update-status.dto.ts`, `find-by-protocol.dto.ts`, `reserva-equipment.mapper.ts`
- Modelo: `src/domain/models/reserva.model.ts`
- Integração com pagamentos: `src/domain/ports/output/payment-refund.port.ts`, `src/infra/adapters/payment/refund-reserve-payment.adapter.ts`
- Erros de domínio: `src/domain/errors.ts`
