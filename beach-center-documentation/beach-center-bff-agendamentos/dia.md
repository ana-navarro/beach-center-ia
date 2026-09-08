# Dia

## Visão geral

O recurso `dia` (`/api/v1/dias`) representa a agenda operacional de uma quadra/unidade em uma
data específica. É exposto pelo microsserviço `beach-center-bff-agendamentos`. Um "dia" agrega o
conjunto de agendamentos (`schedulings`) gerados para aquela data — em slots de 60 minutos, entre
o horário de abertura e fechamento da unidade, pulando o intervalo de almoço quando configurado —
e controla se a data está `opened` (aberta para reservas) ou fechada.

Este recurso concentra a orquestração mais complexa do serviço: criação/geração de agendamentos
sob demanda (`POST /dias`, `POST /dias/:day/schedulings`), fechamento com cancelamento em cascata
de reservas vinculadas (`PATCH /dias/close-date` com `cancel_reserves: true`) e detecção de
conflitos entre agendamentos e eventos recorrentes (`eventos-agendados`).

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /dias` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /dias/close-date` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /dias/open-date` | `authMiddleware` + `requireRole('ADMIN')` |
| `GET /dias` | pública (sem guard) |
| `POST /dias/:day/schedulings` | pública (sem guard) |

`authMiddleware` valida o `idToken` do Firebase enviado em `Authorization: Bearer <token>` e
carrega o usuário espelhado na collection `users`. `requireRole('ADMIN')` exige
`databaseUser.user_type === "ADMIN"` (case-insensitive).

## Endpoints

### POST /api/v1/dias

Cria um ou mais dias, gerando os agendamentos (slots de 60 min) para cada um. Aciona
`CreateDayUsecase`. Para cada item do array:

1. Valida `court`/`unit` como ObjectId e confirma (via `SchedulingReferencesValidator`) que a
   quadra existe e pertence à unidade informada.
2. Gera os slots do dia via `DaySchedulingRulesService.buildSchedulings(item, validateRequestedHours=true)`:
   resolve o horário de funcionamento real da unidade (`SchedulingWindowValidator.resolveOperatingHours`)
   e **valida que `begin_hour`/`end_hour`/`start_lunch_time`/`end_lunch_time` enviados batem
   exatamente** com a configuração da unidade (é `true` apenas neste endpoint — `close-date` e
   `list-day-schedulings` geram os slots sem essa validação estrita).
3. Marca `available: false` nos slots que colidem com um evento `CONFIRMED` do mesmo
   dia-da-semana/quadra/unidade, ou que já estão no passado.
4. Insere apenas os slots que ainda não existem no banco (dedup por `start_time + court + unit`,
   via `findSchedulingsByExactSlotsPort` + `filterMissingSchedulings`).
5. Libera (`available: true`) os agendamentos combinados quando elegível
   (`schedulingAvailabilityService.releaseSchedulingsIfAllowed`).
6. Cria o documento `Day` (se ainda não existir) ou anexa os novos `scheduling_ids` a um dia
   existente.

Se a mesma requisição contiver dois itens duplicados (mesmo `start_time`+`court`+`unit`), a
validação interna `validateDuplicatedSchedulings` rejeita antes de qualquer escrita.

**Corpo da requisição** — array de objetos:

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `day` | string | sim | Data no formato `YYYY-MM-DD` |
| `begin_hour` | string (`HH:mm`) | sim | Horário de abertura — deve bater exatamente com o configurado na unidade |
| `start_lunch_time` | string (`HH:mm`) | não | Início do almoço — deve ser posterior a `begin_hour`; só aceito se a unidade tiver almoço configurado |
| `end_lunch_time` | string (`HH:mm`) | não | Fim do almoço — deve ser posterior a `start_lunch_time` |
| `end_hour` | string (`HH:mm`) | sim | Horário de fechamento — deve ser posterior a `end_lunch_time` (ou `begin_hour`, se não houver almoço); deve bater com o configurado na unidade |
| `court` | string (ObjectId 24-hex) | sim | Quadra |
| `unit` | string (ObjectId 24-hex) | sim | Unidade — deve conter a quadra informada |

O array precisa ter ao menos 1 item (`min(1)`).

**Resposta de sucesso** — `201`:
```json
{
  "message": "Dias criados com sucesso",
  "data": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "day": "2026-09-10T00:00:00.000Z",
      "opened": true,
      "scheduling_ids": ["665f...", "665f..."],
      "schedulings": [
        { "id": "665f...", "date": "2026-09-10T00:00:00.000Z", "start_time": "2026-09-10T11:00:00.000Z", "end_time": "2026-09-10T12:00:00.000Z", "court": "665f...", "unit": "665f...", "available": true }
      ]
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `message: "Dados inválidos"` — falha de validação yup (formato de `day`/hora, ObjectId, sequência cronológica) |
| 400 | `"Informe ao menos um dia para criar"` — array vazio |
| 400 | `"Quadra invalida"` / `"Unidade invalida"` — ObjectId malformado (checado após o yup) |
| 400 | `"Horario de funcionamento invalido para a unidade. Use <begin> ate <end>"` — `begin_hour`/`end_hour` não batem com a configuração da unidade |
| 400 | `"Unidade nao possui horario de almoco configurado"` — enviou `start_lunch_time`/`end_lunch_time` para unidade sem almoço |
| 400 | `"Horario de almoco invalido para a unidade. Use <start> ate <end>"` — almoço enviado diverge do configurado |
| 400 | `"Nenhum agendamento foi gerado para o dia <day>"` — janela resultou em zero slots |
| 400 | `"Payload possui agendamentos duplicados para a mesma data, horario, quadra e unidade"` |
| 404 | `"Quadra nao encontrada"` / `"Unidade nao encontrada"` — via `SchedulingReferencesValidator` |
| 401 | `"Token nao fornecido"` / `"Token invalido ou expirado"` |
| 403 | `"Acesso negado"` — usuário autenticado não é ADMIN |

**Exemplo de chamada**
```bash
curl -X POST "https://<host>/api/v1/dias" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '[{
    "day": "2026-09-10",
    "begin_hour": "08:00",
    "start_lunch_time": "10:00",
    "end_lunch_time": "15:00",
    "end_hour": "23:00",
    "court": "665f1a2b3c4d5e6f7a8b9c0d",
    "unit": "665f1a2b3c4d5e6f7a8b9c0e"
  }]'
```

---

### PATCH /api/v1/dias/close-date

Fecha um dia inteiro ou um recorte específico (quadra+unidade) dele, tornando os agendamentos
afetados indisponíveis. Aciona `CloseDateUsecase`. Este é o endpoint com a orquestração composta
mais sensível do serviço:

1. Se apenas um entre `court`/`unit` for informado, rejeita (`400`) — os dois ou nenhum.
2. Se `court`+`unit` forem informados (fechamento **escopado**), gera/garante os slots daquele
   recorte (mesma lógica de `POST /dias`, porém sem validar os horários enviados — não há campo de
   horário nesta rota) antes de localizar o dia.
3. Busca os agendamentos a fechar (escopados por `court`/`unit`, se informados) e verifica se
   existem **reservas ativas** (`pending`/`approved`) vinculadas a esses agendamentos
   (`findActiveReservesForSchedulingsPort`).
4. **Se houver reservas ativas e `cancel_reserves !== true`**: a operação é abortada e o
   controller retorna o shape especial `CLOSE_DATE_RESERVE_CONFLICT` (ver seção de erros abaixo) —
   nada é fechado.
5. **Se houver reservas ativas e `cancel_reserves === true`**: para cada reserva ativa, o usecase
   chama `DeleteReserveUseCase.execute(reserve.id, { skipCancellationTimeValidation: true })` —
   ou seja, reaproveita exatamente a mesma lógica de cancelamento/estorno do endpoint
   `PATCH /reservas/:id/delete` (ver documentação de `reserva.md`), porém **ignorando a janela de
   2 horas antes do primeiro agendamento**. Se a reserva tiver `payment_id`, o estorno é
   solicitado via `IRefundPaymentPort` (chamada HTTP para `beach-center-bff-pagamentos`) como
   parte dessa chamada reaproveitada.
6. Marca os agendamentos escopados como fechados (`closed_scheduling_ids` do dia) e indisponíveis
   (`available: false`).
7. `opened` do dia só é definido como `false` quando **nem `court` nem `unit`** foram informados
   (fechamento do dia inteiro). Em um fechamento escopado, `opened` permanece com o valor anterior
   — só o subconjunto de agendamentos daquela quadra/unidade entra em `closed_scheduling_ids`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `day` | string | sim | Data no formato `YYYY-MM-DD` |
| `court` | string (ObjectId 24-hex) | não | Escopa o fechamento a uma quadra — exige `unit` junto |
| `unit` | string (ObjectId 24-hex) | não | Escopa o fechamento a uma unidade — exige `court` junto |
| `cancel_reserves` | boolean | não | Se `true`, cancela e estorna automaticamente as reservas ativas vinculadas em vez de retornar conflito |

**Resposta de sucesso** — `200`:
```json
{
  "message": "Dia fechado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "day": "2026-09-10T00:00:00.000Z",
    "opened": false,
    "scheduling_ids": ["665f...", "665f..."],
    "schedulings": [
      { "id": "665f...", "start_time": "...", "end_time": "...", "available": false }
    ]
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `message: "Dados inválidos"` — falha de validação yup |
| 400 | `"Informe unidade e quadra para fechar um recorte especifico"` — apenas `court` ou apenas `unit` informado |
| 404 | `"Nenhum agendamento encontrado para fechar"` — recorte válido mas sem agendamentos |
| 404 | `"Dia nao encontrado"` — dia nunca criado, fechamento sem `court`/`unit` |
| 401 | `"Token nao fornecido"` / `"Token invalido ou expirado"` |
| 403 | `"Acesso negado"` |

**Erro especial — `CLOSE_DATE_RESERVE_CONFLICT` (409)**

Quando existem reservas `pending`/`approved` vinculadas aos agendamentos do recorte e
`cancel_reserves` não foi enviado como `true`, o controller (`close-date.controller.ts`)
intercepta `CloseDateReserveConflictError` **antes** de cair no tratador genérico de erros
(`handle-http-error.ts`) e responde com um shape próprio, diferente do padrão
`{message, errors}` usado no resto do serviço:

```json
{
  "message": "Existem reservas vinculadas aos horarios deste dia. Confirme para cancelar e estornar antes de fechar.",
  "code": "CLOSE_DATE_RESERVE_CONFLICT",
  "reserves": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "number": "1234567890",
      "name": "Fulano de Tal",
      "email": "fulano@example.com",
      "phone": "+5511999999999",
      "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c0e"],
      "total": 160,
      "status": "approved",
      "payment_id": "mock_abc123",
      "refund_status": null
    }
  ]
}
```

| Campo de `reserves[]` | Tipo | Descrição |
|---|---|---|
| `id` | string | ID da reserva |
| `number` | string | Protocolo da reserva |
| `name`, `email`, `phone` | string | Dados de contato informados na reserva |
| `scheduling_id` | string[] | IDs dos agendamentos vinculados a essa reserva |
| `total` | number | Valor total da reserva |
| `status` | string | Status atual (`pending` ou `approved`) |
| `payment_id` | string \| undefined | ID do pagamento, se houver |
| `refund_status` | string \| undefined | Status de um estorno prévio, se houver |

Para reverter esse conflito, o cliente deve reenviar a mesma requisição com
`cancel_reserves: true`.

**Exemplo de chamada**
```bash
curl -X PATCH "https://<host>/api/v1/dias/close-date" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{"day": "2026-09-10", "cancel_reserves": true}'
```

---

### PATCH /api/v1/dias/open-date

Reabre um dia (ou um recorte quadra+unidade) previamente fechado. Aciona `OpenDateUsecase` —
simétrico ao `close-date`, mas **não interage com reservas** (nenhuma reserva é criada/afetada ao
reabrir).

1. Localiza o dia pela data; se não existir, retorna `404`.
2. Busca os agendamentos do escopo (`court`/`unit`, se informados — senão todo o dia).
3. Remove esses IDs de `closed_scheduling_ids` e define `opened: true` no dia (mesmo em uma
   reabertura escopada — diferente do `close-date`, aqui `opened` sempre vira `true`).
4. Libera (`available: true`) os agendamentos elegíveis via
   `schedulingAvailabilityService.releaseSchedulingsIfAllowedStrict` (não libera agendamentos que
   ainda tenham conflito de evento ou reserva ativa).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `day` | string | sim | Data no formato `YYYY-MM-DD` |
| `court` | string (ObjectId 24-hex) | não | Escopa a reabertura a uma quadra |
| `unit` | string (ObjectId 24-hex) | não | Escopa a reabertura a uma unidade |

**Resposta de sucesso** — `200`:
```json
{
  "message": "Dia aberto com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "day": "2026-09-10T00:00:00.000Z",
    "opened": true,
    "scheduling_ids": ["665f...", "665f..."],
    "schedulings": [{ "id": "665f...", "available": true }]
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `message: "Dados inválidos"` — falha de validação yup |
| 404 | `"Dia nao encontrado"` |
| 401 | `"Token nao fornecido"` / `"Token invalido ou expirado"` |
| 403 | `"Acesso negado"` |

**Exemplo de chamada**
```bash
curl -X PATCH "https://<host>/api/v1/dias/open-date" \
  -H "Authorization: Bearer <idToken ADMIN>" \
  -H "Content-Type: application/json" \
  -d '{"day": "2026-09-10"}'
```

---

### GET /api/v1/dias

Lista todos os dias cadastrados. Rota pública. Aciona `ListDaysUsecase`, que antes de listar
marca como indisponíveis (`available: false`) todos os agendamentos cujo horário já passou
(`markPastSchedulingsUnavailablePort`).

**Resposta de sucesso** — `200`:
```json
{
  "message": "Dias listados com sucesso",
  "data": [
    { "id": "665f...", "day": "2026-09-10T00:00:00.000Z", "opened": true, "scheduling_ids": ["665f...", "665f..."] }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 500 | Falha interna inesperada |

**Exemplo de chamada**
```bash
curl -X GET "https://<host>/api/v1/dias"
```

---

### POST /api/v1/dias/:day/schedulings

Lista (e gera sob demanda, se necessário) os agendamentos de uma quadra/unidade em um dia
específico. Rota pública, apesar do verbo `POST` — o corpo é usado apenas para informar
`court`/`unit`. Aciona `ListDaySchedulingsUsecase`, o fluxo mais complexo do recurso:

1. Marca agendamentos passados como indisponíveis (`markPastSchedulingsUnavailablePort`).
2. Valida `court`/`unit` como ObjectId e confirma que a quadra pertence à unidade
   (`SchedulingReferencesValidator`).
3. Busca o dia existente (se houver) e seus agendamentos já persistidos para aquele
   `court`/`unit`.
4. Gera os slots esperados via `DaySchedulingRulesService.buildSchedulings` (sem validar horários
   enviados, pois não há campos de horário nesta rota) e insere no banco apenas os que ainda não
   existem (dedup por `start_time + court + unit` — chamadas repetidas ao mesmo dia/quadra/unidade
   são **idempotentes**, não duplicam agendamentos nem `scheduling_ids`).
5. Cria o dia (se não existia) ou anexa os novos `scheduling_ids`.
6. Para cada agendamento com `available: true`, verifica conflitos com bloqueadores recorrentes
   `CONFIRMED` (`EventConflictService.getConflicts`) e monta a lista `exception_conflicts`. Desde
   a task 004 o `EventConflictService` mescla **três** fontes: `eventos_agendados` (`OUTRO`),
   `mensalista_planos` e `aula_bloqueios`. Cada item em `exceptions[]` traz o campo **`source`**
   (`"EVENT"` | `"MENSALISTA"` | `"AULA"`) indicando a origem do bloqueio; o restante do shape é
   inalterado.
7. Libera agendamentos elegíveis, bloqueia os que têm conflito de evento e os que pertencem a um
   dia (ou recorte) fechado.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `day` | string | sim | Data no formato `YYYY-MM-DD` (path param) |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `court` | string | sim | ID da quadra |
| `unit` | string | sim | ID da unidade |

**Resposta de sucesso** — `200`:
```json
{
  "message": "Agendamentos do dia listados com sucesso",
  "data": [
    { "id": "665f...", "date": "2026-09-10T00:00:00.000Z", "start_time": "...", "end_time": "...", "court": "665f...", "unit": "665f...", "available": true }
  ],
  "exception_conflicts": [
    {
      "scheduling": { "id": "665f...", "start_time": "..." },
      "exceptions": [{ "id": "665f...", "event_type": "MENSALISTA", "day_of_week": "quinta-feira", "start_time": "18:00", "end_time": "20:00", "court": "665f...", "unit": "665f...", "status": "CONFIRMED", "source": "MENSALISTA" }]
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `message: "Dados inválidos"` — `day` fora do formato, `court`/`unit` ausentes |
| 400 | `"Quadra invalido"` / `"Unidade invalido"` — ObjectId malformado |
| 400 | `"Quadra nao pertence a unidade informada"` |
| 404 | `"Quadra nao encontrada"` / `"Unidade nao encontrada"` |
| 404 | `"Dia nao encontrado"` — retornado pelo controller se o usecase devolver `null` |

**Exemplo de chamada**
```bash
curl -X POST "https://<host>/api/v1/dias/2026-09-10/schedulings" \
  -H "Content-Type: application/json" \
  -d '{"court": "665f1a2b3c4d5e6f7a8b9c0d", "unit": "665f1a2b3c4d5e6f7a8b9c0e"}'
```

## Referências

- Rotas: `src/applications/routes/day.route.ts`
- Controllers: `src/applications/controllers/day/create/create-day.controller.ts`,
  `src/applications/controllers/day/close-date/close-date.controller.ts`,
  `src/applications/controllers/day/open-date/open-date.controller.ts`,
  `src/applications/controllers/day/list-days/list-days.controller.ts`,
  `src/applications/controllers/day/list-day-schedulings/list-day-schedulings.controller.ts`
- Usecases: `src/domain/usecases/day/create-day/create-day.usecase.ts`,
  `src/domain/usecases/day/close-date/close-date.usecase.ts`,
  `src/domain/usecases/day/open-date/open-date.usecase.ts`,
  `src/domain/usecases/day/list-days/list-days.usecase.ts`,
  `src/domain/usecases/day/list-day-schedulings/list-day-schedulings.usecase.ts`
- Serviço compartilhado: `src/domain/usecases/day/shared/day-scheduling-rules.service.ts`
- DTOs: `src/applications/dto/create-day.dto.ts`, `src/applications/dto/close-date.dto.ts`,
  `src/applications/dto/open-date.dto.ts`, `src/applications/dto/list-days.dto.ts`,
  `src/applications/dto/list-day-schedulings.dto.ts`
- Modelo: `src/domain/models/day.model.ts`
- Erros: `src/domain/errors.ts` (`CloseDateReserveConflictError`, `ICloseDateReserveConflict`)
- Cross-referência (cancelamento/estorno em cascata): `src/domain/usecases/reserva/delete/delete-reserve.usecase.ts`
