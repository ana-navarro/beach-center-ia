# Agendamento (Scheduling)

## Visão geral

Recurso `scheduling`, exposto pelo microsserviço `beach-center-bff-agendamentos` sob o path
`/api/v1/agendamentos`. Representa um **slot de horário** (data + hora de início/fim) em uma
quadra (`court`) de uma unidade (`unit`), com uma flag `available` que indica se o horário pode
ser reservado. É a entidade central do domínio de agendamentos: `reserva` referencia
`agendamento` por `scheduling_id`, `evento-agendado` bloqueia/libera agendamentos recorrentes, e
`dia` gera lotes de agendamentos para uma data.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /agendamentos` | pública (sem middleware) |
| `GET /agendamentos/:id` | `authMiddleware` + `requireRole('ADMIN')` |
| `GET /agendamentos` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /agendamentos/:id` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /agendamentos/:id/delete` | `authMiddleware` + `requireRole('ADMIN')` |

`requireRole('ADMIN')` exige um `idToken` Firebase válido (`Authorization: Bearer <idToken>`) de
um usuário cujo `user_type` (na coleção `users`, espelhada do serviço `usuarios`) seja `ADMIN`.

## Endpoints

### POST /api/v1/agendamentos

Cria um novo agendamento (slot de horário). Pública — usada tanto pelo fluxo interno (`POST
/dias`, `POST /dias/:day/schedulings`) quanto por integrações externas. Aciona
`CreateSchedulingUsecase`, que orquestra 3 validações de domínio antes de persistir:

1. `SchedulingReferencesValidator` — confirma que `court` e `unit` existem (busca ativa/não
   deletada, em paralelo via `Promise.all`) e que a quadra pertence à unidade informada.
2. `SchedulingWindowValidator` — valida que a data não é passada, que `date`/`start_time`/
   `end_time` pertencem ao mesmo dia, que `end_time > start_time`, que o horário está dentro do
   funcionamento da unidade e fora do intervalo de almoço (ver regras especiais abaixo).
3. `EventConflictService.hasConflict` — verifica se existe um **bloqueador recorrente `CONFIRMED`**
   conflitando com o dia da semana e a janela de horário nessa quadra/unidade. Desde a task 004
   são **3 fontes**: `eventos_agendados` (`OUTRO`), `mensalista_planos` e `aula_bloqueios`.

O campo `available` salvo é sempre `false` se houver conflito com qualquer uma das 3 fontes **ou**
se a data for passada; caso contrário, usa o valor enviado no body (default `false` se omitido).

**Regra especial de horário de funcionamento** (`SchedulingWindowValidator`):
- Unidade com `_id = 6a440a931094fad2f585011b` (`unitOneId`): funcionamento **18:00–22:00**, **sem** intervalo de almoço.
- Qualquer outra unidade: funcionamento **08:00–23:00**, com almoço bloqueado das **10:00 às 15:00**.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `date` | date (ISO) | sim | Data do agendamento |
| `start_time` | string `"HH:mm"` | sim | Hora de início |
| `end_time` | string `"HH:mm"` | sim | Hora de fim; deve ser posterior a `start_time` (validado no DTO) |
| `court` | string (ObjectId 24 hex) | sim | ID da quadra |
| `unit` | string (ObjectId 24 hex) | sim | ID da unidade |
| `available` | boolean | não | Disponibilidade desejada (default `false`; sobrescrito para `false` se houver conflito de evento ou data passada) |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Agendamento criado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "date": "2026-09-10T00:00:00.000Z",
    "start_time": "2026-09-10T09:00:00.000Z",
    "end_time": "2026-09-10T10:00:00.000Z",
    "court": "6a440a931094fad2f585011a",
    "unit": "6a440a931094fad2f585011b",
    "available": false
  }
}
```

**Erros possíveis**

| Código | Causa |
|---|---|
| 400 | `Dados inválidos` — falha de validação yup (campo obrigatório ausente, `court`/`unit` não é ObjectId de 24 caracteres, ou `start_time` não é anterior a `end_time` — mensagem no campo `end_time`: `"start_time deve ser anterior ao end_time"`) |
| 400 | `Nao e permitido criar agendamentos para datas que ja passaram` |
| 400 | `Data e horarios devem pertencer ao mesmo dia` |
| 400 | `Horario final deve ser maior que o horario inicial` (checagem redundante no usecase; na prática o DTO barra `end_time <= start_time` antes) |
| 400 | `Horario fora do funcionamento da unidade. Use 08:00 ate 23:00` (ou `Use 18:00 ate 22:00` para `unitOneId`) |
| 400 | `Horario indisponivel no almoco da unidade. Use antes de 10:00 ou depois de 15:00` |
| 400 | `Quadra nao pertence a unidade informada` |
| 404 | `Quadra nao encontrada` (verificada antes de `unit`, quando ambas faltam) |
| 404 | `Unidade nao encontrada` |

**Exemplo de chamada**

```bash
curl -X POST http://localhost:5000/api/v1/agendamentos \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-09-10",
    "start_time": "09:00",
    "end_time": "10:00",
    "court": "6a440a931094fad2f585011a",
    "unit": "6a440a931094fad2f585011b"
  }'
```

---

### GET /api/v1/agendamentos/:id

Busca um agendamento por ID. Aciona `ReadSchedulingUsecase`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId 24 hex) | sim | ID do agendamento |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "date": "2026-09-10T00:00:00.000Z",
    "start_time": "2026-09-10T09:00:00.000Z",
    "end_time": "2026-09-10T10:00:00.000Z",
    "court": "6a440a931094fad2f585011a",
    "unit": "6a440a931094fad2f585011b",
    "available": true
  }
}
```

**Erros possíveis**

| Código | Causa |
|---|---|
| 400 | `ID inválido` — `id` não é um ObjectId de 24 caracteres hexadecimais |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` — Bearer ausente/inválido |
| 403 | `Acesso negado` — usuário autenticado não é `ADMIN` |
| 404 | `Agendamento não encontrado` |

**Exemplo de chamada**

```bash
curl -X GET http://localhost:5000/api/v1/agendamentos/665f1a2b3c4d5e6f7a8b9c0d \
  -H "Authorization: Bearer <idToken>"
```

---

### GET /api/v1/agendamentos

Lista agendamentos com filtros opcionais. Aciona `ListSchedulingsUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `date` | string | não | Filtra por data exata |
| `unit` | string (ObjectId) | não | Filtra por unidade |
| `court` | string (ObjectId) | não | Filtra por quadra |
| `available` | string `"true"`/`"false"` | não | Filtra por disponibilidade; qualquer outro valor retorna 400. Validação feita **no controller**, não no DTO/yup |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Agendamentos listados com sucesso",
  "data": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "date": "2026-09-10T00:00:00.000Z",
      "start_time": "2026-09-10T09:00:00.000Z",
      "end_time": "2026-09-10T10:00:00.000Z",
      "court": "6a440a931094fad2f585011a",
      "unit": "6a440a931094fad2f585011b",
      "available": true
    }
  ]
}
```

**Erros possíveis**

| Código | Causa |
|---|---|
| 400 | `available deve ser true ou false` — valor de query diferente de `"true"`/`"false"` (verificação feita diretamente no controller `list-scheduling.controller.ts`, antes de chamar o usecase) |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |

**Exemplo de chamada**

```bash
curl -X GET "http://localhost:5000/api/v1/agendamentos?unit=6a440a931094fad2f585011b&available=true" \
  -H "Authorization: Bearer <idToken>"
```

---

### PATCH /api/v1/agendamentos/:id

Atualiza um agendamento existente. Aciona `UpdateSchedulingUsecase`, que reaplica as **mesmas 3
validações** do create (`SchedulingReferencesValidator`, `SchedulingWindowValidator`,
`EventConflictService`) sobre os novos dados. O corpo exige `date`/`start_time`/`end_time`/
`court`/`unit` (substituição completa desses campos, não um PATCH parcial).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId 24 hex) | sim | ID do agendamento |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `date` | date (ISO) | sim | Nova data |
| `start_time` | string `"HH:mm"` | sim | Nova hora de início |
| `end_time` | string `"HH:mm"` | sim | Nova hora de fim (deve ser posterior a `start_time`) |
| `court` | string (ObjectId 24 hex) | sim | Nova quadra |
| `unit` | string (ObjectId 24 hex) | sim | Nova unidade |
| `available` | boolean | não | Disponibilidade desejada (default `false`; sobrescrita para `false` se houver conflito de evento ou data passada) |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Agendamento encontrado com sucesso",
  "data": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "date": "2026-09-11T00:00:00.000Z",
    "start_time": "2026-09-11T09:00:00.000Z",
    "end_time": "2026-09-11T10:00:00.000Z",
    "court": "6a440a931094fad2f585011a",
    "unit": "6a440a931094fad2f585011b",
    "available": true
  },
  "list": [ "...todos os agendamentos atualizados recentemente (IListUpdatedSchedulingsPort)..." ]
}
```

**Erros possíveis**

| Código | Causa |
|---|---|
| 400 | `Dados inválidos` — falha de validação yup |
| 400 | `ID inválido` |
| 400 | `Nao e permitido criar agendamentos para datas que ja passaram` / `Data e horarios devem pertencer ao mesmo dia` / `Horario final deve ser maior que o horario inicial` / `Horario fora do funcionamento da unidade...` / `Horario indisponivel no almoco da unidade...` / `Quadra nao pertence a unidade informada` (mesmas mensagens do create) |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |
| 404 | `Quadra nao encontrada` / `Unidade nao encontrada` |
| 404 | `Agendamento não encontrado` — id não existe, ou `updateSchedulingPort` retorna `null` |

**Exemplo de chamada**

```bash
curl -X PATCH http://localhost:5000/api/v1/agendamentos/665f1a2b3c4d5e6f7a8b9c0d \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "date": "2026-09-11",
    "start_time": "09:00",
    "end_time": "10:00",
    "court": "6a440a931094fad2f585011a",
    "unit": "6a440a931094fad2f585011b"
  }'
```

---

### PATCH /api/v1/agendamentos/:id/delete

Remove (soft delete) um agendamento. Aciona `DeleteSchedulingUsecase`, que bloqueia a exclusão se
houver **qualquer** reserva vinculada (ativa ou não — `IHasReserveForSchedulingPort` não filtra
por status) e, em caso de sucesso, também remove o id do agendamento de `dias.scheduling_ids`
(`IRemoveSchedulingFromDaysPort`, `updateMany $pull`).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId 24 hex) | sim | ID do agendamento |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Agendamento deletado com sucesso",
  "data": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "date": "2026-09-10T00:00:00.000Z",
      "start_time": "2026-09-10T09:00:00.000Z",
      "end_time": "2026-09-10T10:00:00.000Z",
      "court": "6a440a931094fad2f585011a",
      "unit": "6a440a931094fad2f585011b",
      "available": false
    }
  ]
}
```
> `data` é um array (`IScheduling[]`), formato legado preservado do adapter original — não um único objeto.

**Erros possíveis**

| Código | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |
| 404 | `Agendamento não encontrado` |
| 409 | `Nao e possivel deletar um agendamento que possui reserva vinculada` |

**Exemplo de chamada**

```bash
curl -X PATCH http://localhost:5000/api/v1/agendamentos/665f1a2b3c4d5e6f7a8b9c0d/delete \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `src/applications/routes/scheduling.route.ts`
- Controllers: `src/applications/controllers/scheduling/{create,read,list,update,delete}/*.controller.ts`
- Usecases: `src/domain/usecases/scheduling/{create,read,list,update,delete}/*.usecase.ts`
- Validador de janela de funcionamento: `src/domain/usecases/scheduling/shared/scheduling-window.validator.ts`
- Validador de referências (quadra/unidade): `src/domain/usecases/shared/scheduling-references.validator.ts`
- Serviço de conflito de evento: `src/domain/usecases/shared/event-conflict.service.ts`
- DTOs: `src/applications/dto/create-scheduling.dto.ts`, `src/applications/dto/update-scheduling.dto.ts`
- Model: `src/domain/models/scheduling.model.ts`
- Ports: `src/domain/ports/input/scheduling.input-port.ts`, `src/domain/ports/output/scheduling-persistence.port.ts`
- Erros de domínio: `src/domain/errors.ts`
- Middleware de auth: `src/applications/middlewares/auth.middleware.ts`
