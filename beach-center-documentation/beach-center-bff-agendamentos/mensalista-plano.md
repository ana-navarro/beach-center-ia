# Mensalista — Plano

## Visão geral

O recurso `mensalista-plano` é o **agendamento recorrente semanal** de um mensalista numa quadra. É exposto pelo serviço `beach-center-bff-agendamentos` como recurso **aninhado** sob `/api/v1/mensalistas/:mensalista_id/planos` (`Router({ mergeParams: true })`).

Um plano CONFIRMED é um **bloqueador recorrente de quadra**: junto com `eventos_agendados` (`OUTRO`) e `aula_bloqueios`, alimenta o `EventConflictService` (kernel de conflito unificado, task 004). Criar um plano bloqueia os `scheduling`s da quadra nos dias/horário; cancelar libera.

- **`unit` é derivado** da `court` no create (a quadra pertence a exatamente uma unidade — `FindUnitIdByCourtIdAdapter`).
- **`start_time`/`end_time`** são strings `"HH:MM"`, arredondadas com `formatRoundedTime` (start trunca, end arredonda p/ cima), igual a `eventos_agendados`.
- **Campos estruturais (`dias`, `start_time`, `end_time`, `court`) NÃO são editáveis** — para mudá-los, cancele o plano e crie outro. O `PATCH /:id` só altera `modalidade`, `equipamentos_proprios` e `price`.
- `status`: `CONFIRMED` | `CANCELLED`. Só `CANCELLED` libera a quadra.

## Autenticação/autorização

Todos os 5 endpoints exigem `Authorization: Bearer <idToken>` válido **e** `user_type: "ADMIN"` (`authMiddleware` + `requireRole("ADMIN")`).

## Endpoints

### POST /api/v1/mensalistas/:mensalista_id/planos

Cria um plano. Aciona `CreateMensalistaPlanoUsecase`:

1. Valida `mensalista_id` (24 hex) e que o mensalista existe (404 senão).
2. Valida `court` (24 hex) e deriva a `unit` (`FindUnitIdByCourtIdPort`) — 404 `Quadra não encontrada` se a quadra não pertence a nenhuma unidade.
3. Arredonda `start_time`/`end_time` (`formatRoundedTime`).
4. Para **cada dia** em `dias`, checa conflito (`EventConflictService.hasConflict`) contra as 3 fontes recorrentes `CONFIRMED` na mesma quadra/unidade/dia-da-semana/janela. Se algum conflitar → `MensalistaPlanoConflictError` (**400**), nada é persistido.
5. Persiste o plano com `status: "CONFIRMED"`.
6. Para cada dia, chama `EventSchedulingImpactService.applyEventToSchedulings` — marca `available: false` nos `scheduling`s de hoje em diante que batem.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mensalista_id` | string (path) | Sim | ObjectId (24 hex) do mensalista dono do plano |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dias` | string[] | Sim | Dias da semana (texto livre, normalizado). Mínimo 1 |
| `start_time` | string | Sim | `"HH:MM"` |
| `end_time` | string | Sim | `"HH:MM"`, maior que `start_time` |
| `court` | string | Sim | ObjectId (24 hex) da quadra. `unit` é derivado |
| `modalidade` | string | Sim | Texto livre (ex.: "Beach Tênis") |
| `equipamentos_proprios` | boolean | Sim | Informativo (sem regra) |
| `price` | number | Não | Valor da mensalidade, `>= 0` |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Plano de mensalista criado com sucesso",
  "data": {
    "id": "665f...",
    "mensalista_id": "665f...",
    "dias": ["segunda", "quarta"],
    "start_time": "08:00",
    "end_time": "09:00",
    "court": "665f...",
    "unit": "665f...",
    "modalidade": "Beach Tênis",
    "equipamentos_proprios": false,
    "status": "CONFIRMED",
    "price": 200
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `dias` vazio, `start_time`/`end_time` fora de `HH:MM`, `end_time <= start_time`, `court` não é ObjectId, `price < 0` |
| 400 | `Conflito de horario com outro agendamento recorrente na mesma quadra.` (`MensalistaPlanoConflictError`) |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Mensalista não encontrado` / `Quadra não encontrada` |

**Exemplo de chamada**
```bash
curl -X POST http://localhost:5000/api/v1/mensalistas/665f.../planos \
  -H "Authorization: Bearer <idToken>" -H "Content-Type: application/json" \
  -d '{ "dias": ["segunda","quarta"], "start_time": "08:00", "end_time": "09:00", "court": "665f...", "modalidade": "Beach Tênis", "equipamentos_proprios": false, "price": 200 }'
```

---

### GET /api/v1/mensalistas/:mensalista_id/planos

Lista **todos os planos** (qualquer status) do mensalista. Aciona `ListMensalistaPlanosUsecase` (valida `mensalista_id`).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Planos de mensalista listados com sucesso", "data": [ { "id": "665f...", "mensalista_id": "665f...", "dias": ["segunda"], "start_time": "08:00", "end_time": "09:00", "court": "665f...", "unit": "665f...", "modalidade": "Beach Tênis", "equipamentos_proprios": false, "status": "CONFIRMED" } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID de mensalista inválido` |
| 401 / 403 | auth / não-ADMIN |

---

### GET /api/v1/mensalistas/:mensalista_id/planos/:id

Busca um plano pelo `id`. Aciona `ReadMensalistaPlanoUsecase` (valida `id`; 404 senão). O `:mensalista_id` do path não é revalidado contra o plano.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Plano de mensalista encontrado com sucesso", "data": { "id": "665f...", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Plano de mensalista não encontrado` |

---

### PATCH /api/v1/mensalistas/:mensalista_id/planos/:id

Atualiza **só campos não estruturais** (`modalidade`, `equipamentos_proprios`, `price`). Aciona `UpdateMensalistaPlanoUsecase`. Campos estruturais enviados no corpo são **ignorados** (`stripUnknown` + o DTO só declara os 3).

**Corpo da requisição** (todos opcionais)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `modalidade` | string | Não | |
| `equipamentos_proprios` | boolean | Não | |
| `price` | number | Não | `>= 0` |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Plano de mensalista atualizado com sucesso", "data": { "id": "665f...", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (`price < 0`) / `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Plano de mensalista não encontrado` |

---

### PATCH /api/v1/mensalistas/:mensalista_id/planos/:id/delete

Cancela o plano. Aciona `CancelMensalistaPlanoUsecase`:

1. Lê o plano; 404 se não existe.
2. `status → "CANCELLED"`.
3. **Se estava `CONFIRMED`**, para cada dia chama `EventSchedulingImpactService.releaseEventFromSchedulings` — libera os `scheduling`s sem reserva ativa e não bloqueados por outro bloqueador recorrente `CONFIRMED`. Se já estava `CANCELLED`, nada acontece.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Plano de mensalista cancelado com sucesso", "data": { "id": "665f...", "status": "CANCELLED", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Plano de mensalista não encontrado` |

**Exemplo de chamada**
```bash
curl -X PATCH http://localhost:5000/api/v1/mensalistas/665f.../planos/665f.../delete -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `src/applications/routes/mensalista-plano.route.ts` (montada em `routes.ts` como `/mensalistas/:mensalista_id/planos`, **antes** de `/mensalistas`)
- Controllers: `src/applications/controllers/mensalista_plano/{create,read,list,update,cancel}/*.controller.ts`
- Usecases: `src/domain/usecases/mensalista-plano/{create,read,list,update,cancel}/*.usecase.ts` + `shared/mensalista-plano-conflict.error.ts`
- Serviços de domínio compartilhados: `src/domain/usecases/shared/event-conflict.service.ts`, `src/domain/usecases/shared/event-scheduling-impact.service.ts`
- Adapters: `src/infra/adapters/mensalista_plano/{create,read,list,update,set-status,cancel-by-mensalista,find-confirmed-by-court-unit}/*.adapter.ts` + `src/infra/adapters/unit/find-id-by-court/find-unit-id-by-court.adapter.ts`
- Schema: `src/infra/schemas/mensalista-plano.schema.ts` (coleção `mensalista_planos`)
- DTO: `src/applications/dto/mensalista-plano.dto.ts`
- Modelo: `src/domain/models/mensalista-plano.model.ts`
- Ports: `src/domain/ports/input/mensalista-plano.input-port.ts`, `src/domain/ports/output/mensalista-plano-persistence.port.ts`, `src/domain/ports/output/mensalista-plano-shared.port.ts`
- Utilitários de data/hora: `src/shared/date-time.ts` (`formatRoundedTime`)
