# Reserva

## Visão geral

O recurso `reserva` (`/api/v1/reservas`) representa a reserva de um ou mais horários (`agendamentos`) por um cliente, incluindo protocolo público e o ciclo de vida de status estilo e-commerce. É exposto pelo serviço **beach-center-bff-agendamentos**.

**Máquina de estados (task 006a):**

```
pending  →  waiting_approve  →  approved
   │              │
   │              └──────────────→  rejected
   └──────────────────────────────→  expired   (24 h sem comprovante)
qualquer estado não-terminal  ──────→  cancelled
```

- `pending` — reserva criada, **sem comprovante e sem protocolo** (`number`). Bloqueia os agendamentos.
- `waiting_approve` — o comprovante foi enviado (em `beach-center-bff-pagamentos`); **é aqui que o protocolo (`number`) é gerado**, com unicidade global (reservas + `ranking_agendamentos`). Continua bloqueando os agendamentos.
- `approved` — o admin confirmou o comprovante. Bloqueia os agendamentos.
- `rejected` / `cancelled` / `expired` — terminais; **liberam** os agendamentos (se nada mais os bloquear).
- `expired` — a varredura lazy do `GET /reservas` marca assim toda reserva `pending` com mais de 24 h de `createdAt` sem comprovante.

**Nota de integração cross-serviço:** três destes endpoints não são consumidos pelo frontend, e sim pelo **beach-center-bff-pagamentos**, autenticando-se com uma chave interna (`x-api-key`) em vez de um usuário Firebase:

- `PATCH /reservas/:id/payment` — `pagamentos` grava aqui os metadados do pagamento (`payment_id`, `checkout_id`, `payment_method`) após um checkout ou webhook Getnet.
- `PATCH /reservas/:id/status` — chamado por `pagamentos` no envio do comprovante (`pending → waiting_approve`) e na revisão do admin (`waiting_approve → approved/rejected`); também pelo webhook Getnet. Bloqueia/libera os agendamentos vinculados.
- `GET /reservas/:id` — `pagamentos` lê os dados da reserva durante o checkout/comprovante, sem Bearer, só com `x-api-key`.

> **Reembolso automático removido (task 006a):** `agendamentos` **não chama mais** `beach-center-bff-pagamentos`. Ao cancelar uma reserva `approved` com valor, apenas grava `refund_status: "manual"` — a devolução do valor é tratada por contato direto entre usuário e admin.

> **Fluxo público por protocolo (task 006b):** além de consultar e cancelar, o cliente pode **reagendar** a reserva pelo protocolo (`PATCH /reservas/protocol/:number/reagendar`). Cancelamento e reagendamento só existem **a partir de `waiting_approve`** — enquanto a reserva está `pending` (sem comprovante), os três endpoints `protocol/*` respondem `404`. Ambos respeitam a **janela de 2 h** antes do agendamento mais cedo. Ao cancelar/reagendar uma reserva `approved`, `agendamentos` dispara uma notificação **não-bloqueante** por WhatsApp (`beach-center-whatsapp`, `POST /messages/send-by-key`) — uma falha de envio nunca reverte a operação já persistida.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /reservas` | `authMiddleware` (Bearer — qualquer usuário autenticado) |
| `GET /reservas/protocol/:number` | pública |
| `PATCH /reservas/protocol/:number/cancel` | pública |
| `PATCH /reservas/protocol/:number/reagendar` | pública |
| `PATCH /reservas/:id/payment` | `internalApiKeyMiddleware` (somente `x-api-key`) |
| `PATCH /reservas/:id/delete` | `authMiddleware` + `requireRole('ADMIN')` |
| `GET /reservas/:id` | `adminOrInternalApiKey` (Bearer+ADMIN **ou** `x-api-key`) |
| `GET /reservas` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /reservas/:id` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /reservas/:id/status` | `adminOrInternalApiKey` (Bearer+ADMIN **ou** `x-api-key`) |

`internalApiKeyMiddleware` e `adminOrInternalApiKey` validam o header `x-api-key` contra `AGENDAMENTOS_INTERNAL_API_KEY`. Quando a autenticação falha nessas duas rotas puramente por API key (`internalApiKeyMiddleware`), a resposta é `401 {"message": "Unauthorized"}` (em inglês, escrita própria do middleware — **não** passa por `handle-http-error.ts`, que produz mensagens em PT-BR).

## Endpoints

### POST /api/v1/reservas

Cria uma reserva para 1 a 3 horários (`scheduling_id`). Aciona `CreateReserveUsecase`, que delega toda a validação a `ReserveSchedulingValidator.validateForReserve`: limite de 3 horários e duplicidade (redundante com a validação do DTO), existência dos agendamentos, data não passada, ausência de conflito com evento agendado (`EventConflictService`), e disponibilidade (`available === true`). Persiste com `status: "pending"` e **sem `number`** — o protocolo só é gerado quando a reserva vai para `waiting_approve` (envio do comprovante). Um `number` explícito no corpo é preservado (caso legado), mas não é o fluxo normal.

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
| `number` | string | Não | Protocolo customizado (caso legado). No fluxo normal **não é enviado** — a reserva nasce sem protocolo |

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
    "equipment": { "self_equipment": true }
  }
}
```

> A reserva recém-criada **não tem `number`** — o campo só aparece a partir de `waiting_approve`.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | DTO inválido (`Dados inválidos`) — campo obrigatório ausente, `scheduling_id` com mais de 3 itens ou repetido, `equipment` inconsistente |
| 400 | `"E permitido selecionar no maximo 3 horarios"` / `"Nao e permitido repetir horarios na mesma reserva"` (redundante com o DTO, mas também validado no usecase) |
| 400 | `"Nao e permitido criar reserva para datas que ja passaram"` |
| 404 | `{"message": "Agendamento não encontrado", "id": [<scheduling_id enviados>]}` — shape especial, **não** segue o padrão `{message}` de `handle-http-error.ts` — ocorre quando algum `scheduling_id` não existe |
| 409 | `"Um ou mais horarios selecionados possuem excecao de agendamento"` — algum horário coincide com um bloqueador recorrente `CONFIRMED` (evento `OUTRO`, `mensalista_plano` ou `aula_bloqueio` — task 004) |
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

Busca uma reserva pelo protocolo público (`number`). Aciona `FindReserveByProtocolUsecase`, que além dos dados da reserva calcula as flags de janela: `can_cancel` / `cancellation_deadline` **e** `can_reschedule` / `reschedule_deadline` (task 006b — cancelar e reagendar usam a **mesma** regra: 2h antes do agendamento mais cedo vinculado, limite exato inclusive `<=`). As duas flags carregam sempre o mesmo valor. Reservas em estado terminal (`cancelled`, `rejected`, `expired`) ou sem agendamentos vinculados retornam ambas `false`; o `*_deadline` é omitido quando não há como calculá-lo.

> **Task 006b:** enquanto a reserva está `pending` (sem comprovante), este endpoint responde **`404`** (`{"message": "Reserva nao encontrada"}`) — a página pública de protocolo só existe a partir de `waiting_approve`.

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
    "status": "waiting_approve",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c01"],
    "can_cancel": true,
    "cancellation_deadline": "2026-09-08T16:00:00.000Z",
    "can_reschedule": true,
    "reschedule_deadline": "2026-09-08T16:00:00.000Z"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `number` ausente/vazio no path (`"Numero de protocolo e obrigatorio"`, shape `{message: "Dados invalidos", errors: [...]}`) |
| 404 | `{"message": "Reserva nao encontrada"}` — protocolo inexistente **ou** reserva ainda em `pending` |

**Exemplo de chamada**

```bash
curl "https://<host>/api/v1/reservas/protocol/1234567890"
```

---

### PATCH /api/v1/reservas/protocol/:number/cancel

Cancela uma reserva pelo protocolo, sem autenticação (fluxo público, ex.: cliente cancelando pelo link recebido). Aciona `DeleteReserveUsecase.executeByProtocol`, que localiza a reserva pelo protocolo, retorna `404` se ainda estiver `pending` (B-btn — task 006b), bloqueia se já estiver `cancelled` ou `expired`, e delega a `execute(id, { motivo })` — mesma lógica de `PATCH /reservas/:id/delete` (janela de 2h; `refund_status: "manual"` se a reserva estava `approved` com valor).

Quando a reserva cancelada estava **`approved`**, dispara — de forma **não-bloqueante** — o WhatsApp `cancelamento-reembolso` (`nome`, `protocolo` e `motivo` quando informado) via `beach-center-whatsapp`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `number` | string | Sim | Número do protocolo da reserva |

**Corpo da requisição** (opcional — o frontend atual cancela **sem** corpo)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `motivo` | string | Não | Motivo do cancelamento (`.trim()`, máx. 280). Registrado no log e repassado ao template de WhatsApp quando presente |

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
| 400 | `number` ausente/vazio (`"Numero de protocolo e obrigatorio"`) / `motivo` acima de 280 caracteres (`"Dados invalidos"`) |
| 400 | `"Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"` |
| 404 | `{"message": "Reserva nao encontrada"}` — protocolo inexistente **ou** reserva ainda em `pending` |
| 409 | `{"message": "Reserva ja foi cancelada"}` — reserva já estava `cancelled` |
| 409 | `{"message": "Reserva expirada"}` — reserva já estava `expired` |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/protocol/1234567890/cancel" \
  -H "Content-Type: application/json" \
  -d '{"motivo": "Imprevisto"}'
```

---

### PATCH /api/v1/reservas/protocol/:number/reagendar

**Rota pública (task 006b).** Reagenda uma reserva pelo protocolo — **refaz a seleção inteira de horários**, mantendo a mesma quantidade (1 a 3). Aciona `RescheduleReserveByProtocolUsecase`:

1. `motivo` é **obrigatório** (400 senão).
2. Localiza a reserva pelo protocolo (`404` se não existe; `404` se `pending`; `409` `"Reserva não pode mais ser reagendada"` se terminal).
3. Aplica a **janela de 2 h** sobre os agendamentos **atuais** (409 senão).
4. `slots` precisa ter exatamente a mesma quantidade de horários da reserva (409 `"O reagendamento deve manter o mesmo numero de horarios"`).
5. Resolve cada `slot` para um `scheduling` **já existente** (mesma `unit`/`court`/`date`/`start_time`) via `findSchedulingsByExactSlots` — 409 `"Horario indisponivel para reagendamento"` se algum não casar.
6. Revalida os novos horários: limite de 3, não no passado, sem conflito com exceção recorrente `CONFIRMED`, disponíveis, e janela de 2 h em **cada** novo slot.
7. Bloqueia os novos `scheduling`s, re-vincula a reserva (`scheduling_id`), libera os antigos elegíveis, grava auditoria na coleção `reserva_reagendamento_auditorias` (`motivo`, `slots_anteriores[]`, `slots_novos[]`).
8. Dispara — **não-bloqueante** — o WhatsApp `reagendamento-confirmado` (`nome`, `protocolo`, `novo_dia`, `novo_horario`, `quadra` do primeiro novo horário).

> O status da reserva **não muda** — só o conjunto de horários. Pagamento/comprovante seguem como estavam.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `number` | string | Sim | Número do protocolo da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `motivo` | string | Sim | `.trim()`, 1 a 280 caracteres — gravado na auditoria |
| `slots` | object[] | Sim | 1 a 3 itens, **mesma quantidade** de horários da reserva atual |
| `slots[].unit` | string | Sim | ObjectId (24 hex) da unidade |
| `slots[].court` | string | Sim | ObjectId (24 hex) da quadra |
| `slots[].date` | string | Sim | `"YYYY-MM-DD"` |
| `slots[].start_time` | string | Sim | `"HH:MM"` |
| `slots[].end_time` | string | Sim | `"HH:MM"` |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reserva reagendada com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "number": "1234567890",
    "status": "approved",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c99"],
    "...": "..."
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `"Dados invalidos"` (DTO — `motivo`/`slots` inválidos) |
| 400 | `"Motivo é obrigatório para reagendar"` |
| 400 | `"Horario invalido no reagendamento"` — `start_time`/`end_time` fora de `HH:MM` |
| 400 | `"Nao e permitido criar reserva para datas que ja passaram"` |
| 404 | `{"message": "Reserva nao encontrada"}` — protocolo inexistente **ou** reserva em `pending` |
| 409 | `"Reserva não pode mais ser reagendada"` — reserva em estado terminal |
| 409 | `"Reserva so pode ser reagendada ate 2 horas antes do primeiro agendamento"` |
| 409 | `"O reagendamento deve manter o mesmo numero de horarios"` |
| 409 | `"Horario indisponivel para reagendamento"` — nenhum `scheduling` corresponde ao slot pedido |
| 409 | `"Novo horario deve comecar ao menos 2 horas a partir de agora"` |
| 409 | `"Um ou mais horarios selecionados possuem excecao de agendamento"` / `"Um ou mais horarios selecionados nao estao disponiveis"` |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/reservas/protocol/1234567890/reagendar" \
  -H "Content-Type: application/json" \
  -d '{
    "motivo": "Conflito de agenda do cliente",
    "slots": [
      { "unit": "665d1a2b3c4d5e6f7a8b9c00", "court": "665c1a2b3c4d5e6f7a8b9c01", "date": "2026-09-20", "start_time": "18:00", "end_time": "19:00" }
    ]
  }'
```

---

### PATCH /api/v1/reservas/:id/payment

**Consumido por `beach-center-bff-pagamentos`.** Atualiza os metadados de pagamento de uma reserva (chamado após checkout / webhook Getnet, e no review manual para gravar `payment_method: "PIX"`). Aciona `UpdateReservePaymentMetadataUsecase`, que apenas valida o `id` e delega à persistência — não há regra de negócio adicional. Os campos `refund_*` continuam aceitos pelo DTO por compatibilidade, mas o fluxo de reembolso automático foi removido (task 006a).

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
| `refund_id` | string | Não | (legado) Identificador do estorno |
| `refund_status` | string | Não | (legado) `"pending"` \| `"approved"` \| `"failed"` \| `"skipped"` \| `"manual"` |
| `refunded_at` | string (data) | Não | (legado) Data/hora do estorno |

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

Cancela (soft) uma reserva por ID, restrito a ADMIN. Aciona `DeleteReserveUsecase.execute`: valida que os agendamentos vinculados ainda existem, aplica a janela de cancelamento de 2h (a menos que chamado internamente com `skipCancellationTimeValidation`, usado pelo fluxo `PATCH /dias/close-date` com `cancel_reserves: true`), grava `refund_status: "manual"` se a reserva estiver `approved` com valor > 0 (**sem** chamada externa de estorno — task 006a), e libera os agendamentos elegíveis (`SchedulingAvailabilityService.releaseSchedulingsIfAllowed`).

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

Lista reservas com filtros opcionais, restrito a ADMIN. Aciona `ListReservesUsecase`, que a cada chamada:

1. **Roda a varredura de expiração** (`ExpirePendingReservesUsecase`): toda reserva `pending` com `createdAt` anterior a `agora − 24 h` (e sem comprovante) passa a `expired` e tem seus agendamentos liberados. Não há cron — a varredura é lazy, disparada por esta listagem.
2. **Ordena os `waiting_approve` primeiro** (o admin precisa ver o que ainda não revisou); a ordem relativa das demais é preservada.
3. **Devolve `pending_approval_count`** — quantas reservas do resultado (após a varredura e o filtro) estão em `waiting_approve`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Não | `"pending"` \| `"waiting_approve"` \| `"approved"` \| `"rejected"` \| `"cancelled"` \| `"expired"` |
| `name` | string | Não | Filtro por nome |
| `email` | string | Não | Filtro por e-mail |
| `phone` | string | Não | Filtro por telefone |
| `scheduling_id` | string | Não | Filtro por horário vinculado |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Reservas listadas com sucesso",
  "data": [
    { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "waiting_approve", "number": "0348871290", "...": "..." },
    { "id": "665f1a2b3c4d5e6f7a8b9c0e", "status": "pending", "...": "..." }
  ],
  "pending_approval_count": 1
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 / 403 | Sem token / não-ADMIN |

**Exemplo de chamada**

```bash
curl "https://<host>/api/v1/reservas?status=waiting_approve" \
  -H "Authorization: Bearer <idToken ADMIN>"
```

---

### PATCH /api/v1/reservas/:id

Atualização parcial de uma reserva, restrito a ADMIN. Aciona `UpdateReserveUsecase`: faz merge dos campos enviados sobre a reserva atual, revalida limite de 3 horários, existência, data não passada e conflito com evento (a menos que o novo `status` seja um terminal que libera — `cancelled`/`rejected`/`expired`), e — quando os `scheduling_id` mudam — valida disponibilidade **apenas dos horários adicionados** (não revalida os que já estavam na reserva). Para `cancelled`/`rejected`/`expired` libera os horários; para os demais, libera os removidos e bloqueia os atuais.

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
| `status` | string | Não | `"pending"` \| `"waiting_approve"` \| `"approved"` \| `"rejected"` \| `"cancelled"` \| `"expired"` |
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

**Consumido por `beach-center-bff-pagamentos`.** É a rota por onde `pagamentos` transiciona o status da reserva: `SubmitManualPaymentUsecase` chama com `waiting_approve` ao receber o comprovante; `ReviewManualPaymentUsecase` chama com `approved`/`rejected` na revisão do admin; o webhook Getnet também a usa. Também acessível a ADMIN via Bearer. Aciona `UpdateReserveAndSchedulingStatusUsecase`:

- Revalida os horários vinculados quando o novo status **bloqueia** (`pending`/`waiting_approve`/`approved`): existência, data não passada, sem conflito de evento, e — se a reserva vinha de um terminal que libera (`cancelled`/`rejected`/`expired`) — disponibilidade.
- **Na transição `→ waiting_approve`**, se a reserva ainda não tem `number`, **gera o protocolo** (10 dígitos, `nanoid`) com unicidade global (consulta `reservas.number` + `ranking_agendamentos.numero_protocolo` via `IIsProtocolNumberTakenPort`) e persiste. É idempotente: se já há `number`, não regenera.
- Bloqueia os agendamentos para `pending`/`waiting_approve`/`approved`; libera-os para `cancelled`/`rejected`/`expired`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string | Sim | ObjectId (24 hex) da reserva |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `status` | string | Sim | `"pending"` \| `"waiting_approve"` \| `"approved"` \| `"rejected"` \| `"cancelled"` \| `"expired"` |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Status atualizado com sucesso",
  "data": {
    "reserve": { "id": "665f1a2b3c4d5e6f7a8b9c0d", "status": "waiting_approve", "number": "0348871290", "...": "..." },
    "scheduling": [{ "id": "665f1a2b3c4d5e6f7a8b9c01", "available": false, "...": "..." }]
  }
}
```

> Ao transicionar para `waiting_approve`, o `reserve.number` retornado é o protocolo recém-gerado — `pagamentos` usa esse valor como prefixo do arquivo do comprovante e como `reserve_number` do registro.

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
  -d '{"status": "waiting_approve"}'
```

## Referências

- Rota: `src/applications/routes/reserva.route.ts`
- Controllers: `src/applications/controllers/reserva/create/create-reserve.controller.ts`, `.../find-by-protocol/find-by-protocol.controller.ts`, `.../cancel-by-protocol/cancel-reserve-by-protocol.controller.ts`, `.../reschedule-by-protocol/reschedule-reserve-by-protocol.controller.ts`, `.../payment-metadata/update-payment-metadata.controller.ts`, `.../delete/delete-reserve.controller.ts`, `.../read/read-reserve.controller.ts`, `.../list/list-reserves.controller.ts`, `.../update/update-reserve.controller.ts`, `src/applications/controllers/shared/update-reserve-and-scheduling-status.controller.ts`
- Usecases: `src/domain/usecases/reserva/create/create-reserve.usecase.ts`, `.../find-by-protocol/find-by-protocol.usecase.ts`, `.../delete/delete-reserve.usecase.ts`, `.../reschedule-by-protocol/reschedule-reserve-by-protocol.usecase.ts`, `.../update-payment-metadata/update-payment-metadata.usecase.ts`, `.../read/read-reserve.usecase.ts`, `.../list/list-reserves.usecase.ts`, `.../update/update-reserve.usecase.ts`, `.../shared/expire-pending-reserves.usecase.ts`, `src/domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts`
- Regras compartilhadas: `src/domain/usecases/shared/reserve-scheduling.validator.ts`, `.../reserve-cancellation-window.ts`, `.../agendamento-time-window.ts` (janela de 2 h reutilizável — task 006b), `.../scheduling-availability.service.ts`, `.../event-conflict.service.ts`
- Protocolo tardio: `src/domain/ports/output/protocol-uniqueness.port.ts`, `src/infra/adapters/protocol/is-protocol-number-taken/`, `src/infra/adapters/reserva/set-protocol-number/set-reserve-protocol-number.adapter.ts`
- Reagendamento por protocolo (task 006b): `src/domain/ports/output/reserve-persistence.port.ts` (`ISetReserveSchedulingsPort`), `src/infra/adapters/reserva/set-schedulings/set-reserve-schedulings.adapter.ts`, `src/domain/models/reserva-reagendamento-auditoria.model.ts`, `src/domain/ports/output/reserva-reagendamento-auditoria-persistence.port.ts`, `src/infra/adapters/reserva_reagendamento_auditoria/create/create-reserva-reagendamento-auditoria.adapter.ts`, `src/infra/schemas/reserva-reagendamento-auditoria.schema.ts` (coleção `reserva_reagendamento_auditorias`)
- WhatsApp (task 006b): `src/domain/ports/output/whatsapp-notification.port.ts`, `src/infra/adapters/whatsapp/send-whatsapp-notification.adapter.ts` (`POST {WHATSAPP_API_URL}/messages/send-by-key`, `x-api-key`; **nunca lança**), env `WHATSAPP_API_URL` / `WHATSAPP_INTERNAL_API_KEY` (`src/config/env.ts`)
- Expiração lazy: `src/infra/adapters/reserva/list-pending-older-than/list-pending-reserves-older-than.adapter.ts`
- DTOs: `src/applications/dto/create-reserva.dto.ts`, `update-reserva.dto.ts`, `update-payment-metadata.dto.ts`, `update-status.dto.ts`, `find-by-protocol.dto.ts`, `cancel-by-protocol.dto.ts`, `reschedule-by-protocol.dto.ts`, `reserva-equipment.mapper.ts`
- Modelo: `src/domain/models/reserva.model.ts`
- Erros de domínio: `src/domain/errors.ts`
- Consumidor externo do WhatsApp: [`../beach-center-whatsapp/mensagem.md`](../beach-center-whatsapp/mensagem.md)
