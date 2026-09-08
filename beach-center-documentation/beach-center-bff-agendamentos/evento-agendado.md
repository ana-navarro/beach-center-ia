# Evento Agendado

## Visão geral

O recurso `evento-agendado` representa exceções recorrentes na grade de horários de uma quadra/unidade — aulas fixas, mensalistas ou qualquer bloqueio semanal (`event_type`: `AULA_BEACH_TENIS`, `MENSALISTA`, `AULA_VOLEI`, `OUTRO`) que ocupam um dia da semana e uma janela de horário específicos. É exposto pelo serviço `beach-center-bff-agendamentos`, sob o path `/api/v1/eventos-agendados`.

Este é o recurso com a orquestração mais crítica do serviço: criar, atualizar ou remover um evento com `status: "CONFIRMED"` **bloqueia ou libera automaticamente** os agendamentos (`scheduling`) futuros da mesma quadra/unidade que caiam no mesmo dia da semana e se sobreponham ao horário do evento, marcando `available: false`/`true` nesses agendamentos. Essa orquestração vive em `EventSchedulingImpactService` (`domain/usecases/shared/event-scheduling-impact.service.ts`), consumida pelos usecases de create/update/delete.

## Autenticação/autorização

Todos os 5 endpoints exigem `Authorization: Bearer <idToken>` válido **e** que o usuário autenticado tenha `user_type: "ADMIN"` (`authMiddleware` + `requireRole("ADMIN")`). Não há endpoint público neste recurso.

## Endpoints

### POST /api/v1/eventos-agendados

Cria um novo evento agendado. Aciona `CreateEventsScheduledUsecase`:

1. Valida que `court` e `unit` existem e que a quadra pertence à unidade informada (`SchedulingReferencesValidator`).
2. Arredonda `start_time`/`end_time` via `formatRoundedTime` — **`start_time` é truncado** (nunca arredonda para cima, mesmo com minutos > 0) e **`end_time` arredonda para cima** quando os minutos são > 0 (ex.: `"20:30"` vira `"21:00"`; `"18:30"` como início vira `"18:00"`).
3. Verifica conflito com outro evento `CONFIRMED` já existente na mesma quadra/unidade/dia da semana/horário sobreposto (`EventConflictService.hasConflict`). Se houver, lança `EventConflictError`.
4. Persiste o evento.
5. Se `status` (default `"CONFIRMED"` quando omitido) for `"CONFIRMED"`, chama `applyEventToSchedulings`: busca todos os agendamentos da mesma quadra/unidade a partir de "hoje" (`getTodayStart()`), filtra os que caem no mesmo dia da semana (normalizado — acento/caixa insensível) e cuja janela de horário se sobrepõe à do evento, e marca `available: false` nesses agendamentos via `ISetSchedulingsAvailabilityPort`. Eventos `CANCELLED` não têm nenhum efeito colateral sobre agendamentos.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `event_type` | string | Sim | Um de `AULA_BEACH_TENIS`, `MENSALISTA`, `AULA_VOLEI`, `OUTRO` |
| `day_of_week` | string | Sim | Nome do dia da semana em texto livre (normalizado internamente — acento/caixa/espaços ignorados) |
| `start_time` | string | Sim | Horário de início, formato `HH:mm` |
| `end_time` | string | Sim | Horário de fim, formato `HH:mm`; deve ser maior que `start_time` |
| `court` | string | Sim | ObjectId (24 hex) da quadra |
| `unit` | string | Sim | ObjectId (24 hex) da unidade |
| `status` | string | Não | `CONFIRMED` (default) ou `CANCELLED` |
| `price` | number | Não | Preço associado ao evento, `>= 0` |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Evento agendado criado com sucesso",
  "data": {
    "id": "665f1c2e4b3a2d1e9f0a1b2c",
    "event_type": "MENSALISTA",
    "day_of_week": "segunda-feira",
    "start_time": "18:00",
    "end_time": "20:00",
    "court": "665f1c2e4b3a2d1e9f0a1111",
    "unit": "665f1c2e4b3a2d1e9f0a2222",
    "status": "CONFIRMED",
    "price": 150
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — corpo não passa na validação yup (`event_type`/`status` fora do enum, `court`/`unit` não são ObjectId de 24 hex, `end_time <= start_time`, campos obrigatórios ausentes) |
| 400 | `Quadra nao pertence a unidade informada` |
| 400 | `Conflito de excecao existente na mesma quadra, unidade e horario.` — **atenção**: apesar de semanticamente ser um conflito, a `EventConflictError` é mapeada para **400**, não 409; comportamento pré-existente preservado deliberadamente |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` — usuário autenticado não é ADMIN |
| 404 | `Quadra nao encontrada` / `Unidade nao encontrada` |

**Exemplo de chamada**

```bash
curl -X POST http://localhost:5000/api/v1/eventos-agendados \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
        "event_type": "MENSALISTA",
        "day_of_week": "segunda-feira",
        "start_time": "18:00",
        "end_time": "20:00",
        "court": "665f1c2e4b3a2d1e9f0a1111",
        "unit": "665f1c2e4b3a2d1e9f0a2222",
        "status": "CONFIRMED"
      }'
```

---

### GET /api/v1/eventos-agendados/:id

Busca um evento agendado pelo `id`. Aciona `ReadEventsScheduledUsecase`, que valida o formato do `id` (24 hex) antes de consultar a persistência.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do evento agendado |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Evento agendado encontrado com sucesso",
  "data": {
    "id": "665f1c2e4b3a2d1e9f0a1b2c",
    "event_type": "MENSALISTA",
    "day_of_week": "segunda-feira",
    "start_time": "18:00",
    "end_time": "20:00",
    "court": "665f1c2e4b3a2d1e9f0a1111",
    "unit": "665f1c2e4b3a2d1e9f0a2222",
    "status": "CONFIRMED",
    "price": 150
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID invalido` — `id` não é um ObjectId de 24 hex válido (mensagem sem acento, diferente do padrão `"ID inválido"` usado em outros recursos) |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |
| 404 | `Evento agendado nao encontrado` |

**Exemplo de chamada**

```bash
curl http://localhost:5000/api/v1/eventos-agendados/665f1c2e4b3a2d1e9f0a1b2c \
  -H "Authorization: Bearer <idToken>"
```

---

### GET /api/v1/eventos-agendados

Lista eventos agendados, com filtros opcionais via query string. Aciona `ListEventsScheduledUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `event_type` | string | Não | Um de `AULA_BEACH_TENIS`, `MENSALISTA`, `AULA_VOLEI`, `OUTRO` |
| `day_of_week` | string | Não | Filtra por dia da semana (comparação exata da string enviada, sem normalização adicional no DTO) |
| `start_time` | string | Não | Filtra por horário de início exato |
| `end_time` | string | Não | Filtra por horário de fim exato |
| `court` | string | Não | ObjectId (24 hex) da quadra |
| `unit` | string | Não | ObjectId (24 hex) da unidade |
| `status` | string | Não | `CONFIRMED` ou `CANCELLED` |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Eventos agendados listados com sucesso",
  "data": [
    {
      "id": "665f1c2e4b3a2d1e9f0a1b2c",
      "event_type": "MENSALISTA",
      "day_of_week": "segunda-feira",
      "start_time": "18:00",
      "end_time": "20:00",
      "court": "665f1c2e4b3a2d1e9f0a1111",
      "unit": "665f1c2e4b3a2d1e9f0a2222",
      "status": "CONFIRMED",
      "price": 150
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `event_type`/`status` fora do enum, ou `court`/`unit` não são ObjectId de 24 hex |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |

**Exemplo de chamada**

```bash
curl "http://localhost:5000/api/v1/eventos-agendados?event_type=MENSALISTA&status=CONFIRMED" \
  -H "Authorization: Bearer <idToken>"
```

---

### PATCH /api/v1/eventos-agendados/:id

Atualiza um evento agendado existente. Aciona `UpdateEventsScheduledUsecase`, que **substitui integralmente** os campos do evento (todos obrigatórios no corpo, exceto `price`) e re-executa a orquestração de bloqueio:

1. Busca o evento existente; se não existir, responde 404 sem validar mais nada.
2. Valida `court`/`unit` (mesmas regras do create).
3. Arredonda `start_time`/`end_time` (mesma lógica do create).
4. Verifica conflito com outro evento `CONFIRMED`, **excluindo o próprio evento sendo atualizado** (`excludeEventId: data.id`) — ou seja, um evento nunca conflita consigo mesmo, mesmo reenviando os mesmos dados.
5. Persiste a atualização.
6. Libera os agendamentos que estavam bloqueados pela **janela antiga** do evento (`releaseEventFromSchedulings` com os dados anteriores) — um agendamento só é liberado se não tiver reserva ativa vinculada e não estiver bloqueado por nenhum outro evento `CONFIRMED` conflitante.
7. Bloqueia os agendamentos que caem na **janela nova** do evento (`applyEventToSchedulings` com os dados atualizados).
8. Retorna o evento atualizado **e** a lista completa de eventos.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do evento agendado |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `event_type` | string | Sim | Um de `AULA_BEACH_TENIS`, `MENSALISTA`, `AULA_VOLEI`, `OUTRO` |
| `day_of_week` | string | Sim | Dia da semana |
| `start_time` | string | Sim | Horário de início `HH:mm` |
| `end_time` | string | Sim | Horário de fim `HH:mm`, maior que `start_time` |
| `court` | string | Sim | ObjectId (24 hex) da quadra |
| `unit` | string | Sim | ObjectId (24 hex) da unidade |
| `status` | string | Sim | `CONFIRMED` ou `CANCELLED` (obrigatório aqui, diferente do create) |
| `price` | number | Não | Preço, `>= 0` |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Evento agendado atualizado com sucesso",
  "data": {
    "id": "665f1c2e4b3a2d1e9f0a1b2c",
    "event_type": "MENSALISTA",
    "day_of_week": "segunda-feira",
    "start_time": "19:00",
    "end_time": "21:00",
    "court": "665f1c2e4b3a2d1e9f0a1111",
    "unit": "665f1c2e4b3a2d1e9f0a2222",
    "status": "CONFIRMED",
    "price": 150
  },
  "list": [
    { "id": "665f1c2e4b3a2d1e9f0a1b2c", "event_type": "MENSALISTA", "...": "..." }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — corpo não passa na validação yup |
| 400 | `Quadra nao pertence a unidade informada` |
| 400 | `Conflito de excecao existente na mesma quadra, unidade e horario.` (não conflita com o próprio evento) |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |
| 404 | `Evento agendado nao encontrado` — evento inexistente, ou `Quadra nao encontrada` / `Unidade nao encontrada` |

**Exemplo de chamada**

```bash
curl -X PATCH http://localhost:5000/api/v1/eventos-agendados/665f1c2e4b3a2d1e9f0a1b2c \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
        "event_type": "MENSALISTA",
        "day_of_week": "segunda-feira",
        "start_time": "19:00",
        "end_time": "21:00",
        "court": "665f1c2e4b3a2d1e9f0a1111",
        "unit": "665f1c2e4b3a2d1e9f0a2222",
        "status": "CONFIRMED"
      }'
```

---

### PATCH /api/v1/eventos-agendados/:id/delete

Remove (hard delete) um evento agendado. Aciona `DeleteEventsScheduledUsecase`:

1. Valida o `id` (24 hex).
2. Remove o evento da persistência; se não existir, retorna `null` e o controller responde 404.
3. Chama `releaseEventFromSchedulings` com os dados do evento removido — libera os agendamentos futuros da mesma quadra/unidade/dia da semana/horário que estavam bloqueados por causa dele, desde que não tenham reserva ativa vinculada e não continuem bloqueados por outro evento `CONFIRMED` conflitante.
4. **Retorna a lista completa de todos os eventos agendados** (via `IListEventsScheduledPort.execute({})`) — não apenas o evento removido. Isso inclui outros eventos `CONFIRMED`/`CANCELLED` que continuam existindo.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do evento agendado a remover |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Evento agendado deletado com sucesso",
  "data": [
    { "id": "665f1c2e4b3a2d1e9f0a9999", "event_type": "AULA_VOLEI", "status": "CONFIRMED", "...": "..." }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID invalido` — `id` não é um ObjectId de 24 hex válido |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` |
| 404 | `Evento agendado nao encontrado` |

**Exemplo de chamada**

```bash
curl -X PATCH http://localhost:5000/api/v1/eventos-agendados/665f1c2e4b3a2d1e9f0a1b2c/delete \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `src/applications/routes/events_scheduled.route.ts`
- Controllers: `src/applications/controllers/events_scheduled/create/create-events-scheduled.controller.ts`, `.../read/read-events-scheduled.controller.ts`, `.../list/list-events-scheduled.controller.ts`, `.../update/update-events-scheduled.controller.ts`, `.../delete/delete-events-scheduled.controller.ts`
- Usecases: `src/domain/usecases/events_scheduled/create/create-events-scheduled.usecase.ts`, `.../read/read-events-scheduled.usecase.ts`, `.../list/list-events-scheduled.usecase.ts`, `.../update/update-events-scheduled.usecase.ts`, `.../delete/delete-events-scheduled.usecase.ts`
- Serviços de domínio compartilhados: `src/domain/usecases/shared/event-conflict.service.ts` (detecção de conflito evento×evento), `src/domain/usecases/shared/event-scheduling-impact.service.ts` (bloqueio/liberação de agendamentos), `src/domain/usecases/shared/scheduling-references.validator.ts` (validação de quadra/unidade)
- Erro de domínio específico: `src/domain/usecases/events_scheduled/shared/event-conflict.error.ts`
- DTOs: `src/applications/dto/events-scheduled.dto.ts`
- Modelo: `src/domain/models/events-scheduled.model.ts`
- Utilitários de data/hora: `src/shared/date-time.ts` (`formatRoundedTime`, `normalizeDayOfWeek`, `getTodayStart`, `timeRangesOverlap`)
- Tratamento de erro HTTP: `src/applications/controllers/shared/handle-http-error.ts`
- Erros de domínio base: `src/domain/errors.ts`
