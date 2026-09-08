# Aula — Bloqueio de quadra (rota interna)

## Visão geral

O recurso `aula-bloqueio` é o **bloqueio recorrente de quadra gerado por uma aula**. É exposto pelo serviço `beach-center-bff-agendamentos` sob `/api/v1/aula-bloqueios`, como **rota interna serviço-a-serviço** — o dono da aula é o `beach-center-bff-aulas` (task 002), que delega o bloqueio de quadra a `agendamentos` via HTTP interno (mesmo padrão `agendamentos → pagamentos`).

Introduzido pela task 004 (que tirou `AULA_BEACH_TENIS`/`AULA_VOLEI` de `eventos_agendados`). Um `aula_bloqueio` CONFIRMED é a terceira fonte do `EventConflictService` (kernel de conflito unificado), junto com `eventos_agendados` (`OUTRO`) e `mensalista_planos`.

- **`unit` é derivado** da `court` no create/update (`FindUnitIdByCourtIdAdapter`).
- **`aula_id`** referencia a aula no `beach-center-bff-aulas`. É **ausente** nos registros criados pela migração de `eventos_agendados` (`npm run migrate:aula-bloqueio`), pois não há aula correspondente.
- **`start_time`/`end_time`** são strings `"HH:MM"` — o `aulas` converte `hora_inicio`/`hora_fim` (`Date`) para `"HH:MM"` no fuso `America/Sao_Paulo` antes de chamar.
- `status`: `CONFIRMED` | `CANCELLED`. Só `CANCELLED` libera a quadra.
- O `PATCH /:id` é um **update estrutural completo** (não parcial): revalida conflito excluindo o próprio bloqueio → persiste → libera o estado antigo → aplica o novo.

## Autenticação/autorização

**Não usa Firebase.** Todos os endpoints exigem o header `x-api-key` igual a `AGENDAMENTOS_INTERNAL_API_KEY` (`internalApiKeyMiddleware`, aplicado com `router.use(...)` para a rota inteira). Sem a chave (ou com chave errada) → **401 `Unauthorized`**. Um ADMIN com token Firebase válido **mas sem `x-api-key`** também recebe 401.

## Endpoints

### POST /api/v1/aula-bloqueios

Cria um bloqueio. Aciona `CreateAulaBloqueioUsecase`:

1. Valida `court` (24 hex) e, se informado, `aula_id` (24 hex).
2. Deriva `unit` da `court` — 404 `Quadra não encontrada` se não pertence a nenhuma unidade.
3. Arredonda `start_time`/`end_time` (`formatRoundedTime`).
4. Para cada dia em `dias`, checa conflito contra as 3 fontes recorrentes `CONFIRMED`. Se conflitar → `AulaBloqueioConflictError` (**409**), nada persistido.
5. Persiste com `status: "CONFIRMED"`.
6. Para cada dia, `applyEventToSchedulings` — bloqueia os `scheduling`s.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dias` | string[] | Sim | Dias da semana (texto livre normalizado). Mínimo 1 |
| `start_time` | string | Sim | `"HH:MM"` |
| `end_time` | string | Sim | `"HH:MM"`, maior que `start_time` |
| `court` | string | Sim | ObjectId (24 hex) da quadra. `unit` é derivado |
| `modalidade` | string | Sim | Texto livre |
| `aula_id` | string | Não | ObjectId (24 hex) da aula no `beach-center-bff-aulas` |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Bloqueio de aula criado com sucesso",
  "data": {
    "id": "665f...",
    "aula_id": "665f...",
    "dias": ["terça"],
    "start_time": "19:00",
    "end_time": "20:00",
    "court": "665f...",
    "unit": "665f...",
    "modalidade": "Beach Tênis",
    "status": "CONFIRMED"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `dias` vazio, `start_time`/`end_time` fora de `HH:MM` ou `end_time <= start_time`, `court`/`aula_id` não são ObjectId |
| 401 | `Unauthorized` — `x-api-key` ausente/errada |
| 404 | `Quadra não encontrada` |
| 409 | `Conflito de horario com outro agendamento recorrente na mesma quadra.` (`AulaBloqueioConflictError`) |

**Exemplo de chamada**
```bash
curl -X POST http://agendamentos:5000/api/v1/aula-bloqueios \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "aula_id": "665f...", "dias": ["terça"], "start_time": "19:00", "end_time": "20:00", "court": "665f...", "modalidade": "Beach Tênis" }'
```

---

### GET /api/v1/aula-bloqueios

Lista bloqueios com filtros opcionais via query. Aciona `ListAulaBloqueiosUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string | Não | ObjectId (24 hex) — filtra pela aula |
| `court` | string | Não | ObjectId (24 hex) — filtra pela quadra |
| `status` | string | Não | `CONFIRMED` ou `CANCELLED` (valores fora disso são ignorados) |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Bloqueios de aula listados com sucesso", "data": [ { "id": "665f...", "aula_id": "665f...", "dias": ["terça"], "start_time": "19:00", "end_time": "20:00", "court": "665f...", "unit": "665f...", "modalidade": "Beach Tênis", "status": "CONFIRMED" } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID de aula inválido` / `ID de quadra inválido` |
| 401 | `Unauthorized` |

---

### GET /api/v1/aula-bloqueios/:id

Busca um bloqueio pelo `id`. Aciona `ReadAulaBloqueioUsecase` (valida `id`; 404 senão).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Bloqueio de aula encontrado com sucesso", "data": { "id": "665f...", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 | `Unauthorized` |
| 404 | `Bloqueio de aula não encontrado` |

---

### PATCH /api/v1/aula-bloqueios/:id

Update **estrutural completo**. Aciona `UpdateAulaBloqueioUsecase`:

1. Valida `id` e `court`; lê o bloqueio (404 senão); deriva a nova `unit`.
2. Arredonda o novo `start_time`/`end_time`.
3. Para cada novo dia, checa conflito **excluindo o próprio bloqueio** (`excludeBloqueioId`). Conflito → `AulaBloqueioConflictError` (409).
4. Persiste o novo estado estrutural.
5. Se o bloqueio estava `CONFIRMED`, libera os `scheduling`s do **estado antigo** (`releaseEventFromSchedulings`).
6. Aplica o **estado novo** (`applyEventToSchedulings`).

`aula_id` e `status` **não** mudam por esta rota.

**Corpo da requisição** (todos obrigatórios)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dias` | string[] | Sim | Mínimo 1 |
| `start_time` | string | Sim | `"HH:MM"` |
| `end_time` | string | Sim | `"HH:MM"`, maior que `start_time` |
| `court` | string | Sim | ObjectId (24 hex). `unit` é rederivado |
| `modalidade` | string | Sim | |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Bloqueio de aula atualizado com sucesso", "data": { "id": "665f...", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — falta um campo estrutural ou formato inválido / `ID inválido` |
| 401 | `Unauthorized` |
| 404 | `Bloqueio de aula não encontrado` / `Quadra não encontrada` |
| 409 | `Conflito de horario com outro agendamento recorrente na mesma quadra.` |

---

### PATCH /api/v1/aula-bloqueios/:id/delete

Cancela o bloqueio identificado por `:id`. Aciona `CancelAulaBloqueioUsecase`. Se a query `?aula_id=<id>` estiver presente, o alvo passa a ser **por aula** (ver abaixo) e o `:id` é ignorado.

Fluxo: lê o bloqueio (404 senão) → `status → "CANCELLED"` → se estava `CONFIRMED`, para cada dia `releaseEventFromSchedulings`.

**Resposta de sucesso** — `200 OK` (o `data` é sempre uma **lista** dos bloqueios cancelados)
```json
{ "message": "Bloqueio(s) de aula cancelado(s) com sucesso", "data": [ { "id": "665f...", "status": "CANCELLED", "...": "..." } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` / `Informe 'id' ou 'aula_id' para cancelar o bloqueio de aula` |
| 401 | `Unauthorized` |
| 404 | `Bloqueio de aula não encontrado` (alvo por `id`) |

---

### PATCH /api/v1/aula-bloqueios/by-aula/:aula_id/delete

Cancela **todos os bloqueios CONFIRMED** de uma aula (usado pelo `beach-center-bff-aulas` em `update-aula` — cancel+recreate — e `delete-aula`). Mesmo usecase (`CancelAulaBloqueioUsecase`), alvo por `aula_id`: lista os CONFIRMED da aula, cancela cada um e libera os `scheduling`s.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (path) | Sim | ObjectId (24 hex) da aula |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Bloqueio(s) de aula cancelado(s) com sucesso", "data": [ { "id": "665f...", "status": "CANCELLED" }, { "id": "665f...", "status": "CANCELLED" } ] }
```
> Se a aula não tem bloqueio CONFIRMED, responde `200` com `data: []` (não é erro).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID de aula inválido` |
| 401 | `Unauthorized` |

**Exemplo de chamada**
```bash
curl -X PATCH http://agendamentos:5000/api/v1/aula-bloqueios/by-aula/665f.../delete \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY"
```

## Referências

- Rota: `src/applications/routes/aula-bloqueio.route.ts` (montada em `routes.ts` como `/aula-bloqueios`; `router.use(internalApiKeyMiddleware)`)
- Middleware: `src/applications/middlewares/internal-api-key.middleware.ts`
- Controllers: `src/applications/controllers/aula_bloqueio/{create,read,list,update,cancel}/*.controller.ts`
- Usecases: `src/domain/usecases/aula-bloqueio/{create,read,list,update,cancel}/*.usecase.ts` + `shared/aula-bloqueio-conflict.error.ts`
- Serviços de domínio compartilhados: `src/domain/usecases/shared/event-conflict.service.ts`, `src/domain/usecases/shared/event-scheduling-impact.service.ts`
- Adapters: `src/infra/adapters/aula_bloqueio/{create,read,list,update-structural,set-status,find-confirmed-by-court-unit}/*.adapter.ts` + `src/infra/adapters/unit/find-id-by-court/*.adapter.ts`
- Schema: `src/infra/schemas/aula-bloqueio.schema.ts` (coleção `aula_bloqueios`)
- DTO: `src/applications/dto/aula-bloqueio.dto.ts`
- Modelo: `src/domain/models/aula-bloqueio.model.ts`
- Ports: `src/domain/ports/input/aula-bloqueio.input-port.ts`, `src/domain/ports/output/aula-bloqueio-persistence.port.ts`, `src/domain/ports/output/aula-bloqueio-shared.port.ts`
- Consumidor (outro serviço): `beach-center-bff-aulas` — ver [`aula.md`](../beach-center-bff-aulas/aula.md)
- Migração de legados: `src/scripts/migrate-aula-bloqueio-from-events.ts` (`npm run migrate:aula-bloqueio`)
