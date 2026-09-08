# Mensalista

## Visão geral

O recurso `mensalista` é o **cadastro de contato** de um cliente que reserva mensalmente quadras da Beach Center. É exposto pelo serviço `beach-center-bff-agendamentos`, sob o path `/api/v1/mensalistas`.

Introduzido pela task 004 (que tirou o `event_type: "MENSALISTA"` de `eventos_agendados`). Um `mensalista` **não tem login** — é só nome, telefone e a data de vencimento da fatura, com controle manual (sem integração com pagamentos). Um mensalista tem 1→N planos de agendamento, geridos pelo recurso aninhado [`mensalista-plano`](./mensalista-plano.md) (`/api/v1/mensalistas/:mensalista_id/planos`).

- **`ativo`** é **derivado** de `vencimento_fatura`: `ativo = vencimento_fatura >= hoje`. O valor é calculado no `create`, recalculado no `update` quando `vencimento_fatura` muda, e **recalculado em lote no `GET` da listagem** (mesmo padrão de `markPastSchedulingsUnavailable`) — a listagem sempre reflete o estado consistente.
- **Inadimplência (`ativo: false`) NÃO libera quadra.** O campo é informativo; só o cancelamento do plano (`PATCH .../planos/:id/delete`) libera os `scheduling`s.
- **Soft-delete manual** (`deleted: true` + filtro `{ deleted: { $ne: true } }` nos adapters). Deletar o mensalista **cancela todos os seus planos** e libera as quadras.

## Autenticação/autorização

Todos os 5 endpoints exigem `Authorization: Bearer <idToken>` válido **e** `user_type: "ADMIN"` (`authMiddleware` + `requireRole("ADMIN")`).

## Endpoints

### POST /api/v1/mensalistas

Cria um mensalista. Aciona `CreateMensalistaUsecase`: deriva `ativo` de `vencimento_fatura >= new Date()` e persiste.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Sim | Nome do mensalista |
| `telefone` | string | Sim | Telefone de contato (sem máscara obrigatória) |
| `vencimento_fatura` | string \| date | Sim | Data de vencimento da fatura (ISO ou `YYYY-MM-DD`). Define `ativo` |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Mensalista criado com sucesso",
  "data": {
    "id": "665f1c2e4b3a2d1e9f0a1b2c",
    "nome": "João da Silva",
    "telefone": "11999990000",
    "vencimento_fatura": "2099-12-31T00:00:00.000Z",
    "ativo": true
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `nome`/`telefone` ausentes ou `vencimento_fatura` não é uma data válida |
| 401 | `Token nao fornecido` / `Token invalido ou expirado` |
| 403 | `Acesso negado` — não é ADMIN |

**Exemplo de chamada**
```bash
curl -X POST http://localhost:5000/api/v1/mensalistas \
  -H "Authorization: Bearer <idToken>" -H "Content-Type: application/json" \
  -d '{ "nome": "João da Silva", "telefone": "11999990000", "vencimento_fatura": "2099-12-31" }'
```

---

### GET /api/v1/mensalistas

Lista **todos** os mensalistas não deletados. Aciona `ListMensalistasUsecase`, que **primeiro** recalcula em lote o `ativo` de todos (`RecalculateMensalistasStatusAdapter` — 2 `updateMany`: vencidos com `ativo:true` → `false`; em dia com `ativo:false` → `true`) e **depois** lista.

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Mensalistas listados com sucesso",
  "data": [
    { "id": "665f...", "nome": "João da Silva", "telefone": "11999990000", "vencimento_fatura": "2099-12-31T00:00:00.000Z", "ativo": true }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 / 403 | auth / não-ADMIN |
| 500 | `Erro interno no servidor` — falha inesperada de persistência |

**Exemplo de chamada**
```bash
curl http://localhost:5000/api/v1/mensalistas -H "Authorization: Bearer <idToken>"
```

---

### GET /api/v1/mensalistas/:id

Busca um mensalista pelo `id`. Aciona `ReadMensalistaUsecase` (valida `id` 24 hex; 404 se não existe/está deletado).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do mensalista |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Mensalista encontrado com sucesso",
  "data": { "id": "665f...", "nome": "João da Silva", "telefone": "11999990000", "vencimento_fatura": "2099-12-31T00:00:00.000Z", "ativo": true }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Mensalista não encontrado` |

---

### PATCH /api/v1/mensalistas/:id

Atualiza **parcialmente** um mensalista. Aciona `UpdateMensalistaUsecase`. Só os campos enviados são alterados; se `vencimento_fatura` mudar, o adapter recalcula `ativo` na hora (`ativo = vencimento_fatura >= hoje`).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do mensalista |

**Corpo da requisição** (todos opcionais)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Não | Novo nome |
| `telefone` | string | Não | Novo telefone |
| `vencimento_fatura` | string \| date | Não | Nova data de vencimento — recalcula `ativo` |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Mensalista atualizado com sucesso", "data": { "id": "665f...", "nome": "João da Silva", "telefone": "11888887777", "vencimento_fatura": "2099-12-31T00:00:00.000Z", "ativo": true } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` / `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Mensalista não encontrado` |

**Exemplo de chamada**
```bash
curl -X PATCH http://localhost:5000/api/v1/mensalistas/665f... \
  -H "Authorization: Bearer <idToken>" -H "Content-Type: application/json" \
  -d '{ "telefone": "11888887777" }'
```

---

### PATCH /api/v1/mensalistas/:id/delete

Soft-delete do mensalista **e cascade**. Aciona `DeleteMensalistaUsecase`:

1. Marca `deleted: true` no mensalista; 404 se não existir.
2. Cancela **todos os planos CONFIRMED** do mensalista (`status: "CANCELLED"`).
3. Para cada dia de cada plano cancelado, chama `EventSchedulingImpactService.releaseEventFromSchedulings` — libera os `scheduling`s da quadra/horário que não têm reserva ativa e não estão bloqueados por outro bloqueador recorrente `CONFIRMED` (evento `OUTRO`, outro plano ou `aula_bloqueio`).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do mensalista |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Mensalista removido com sucesso", "data": { "id": "665f...", "nome": "João da Silva", "telefone": "11888887777", "vencimento_fatura": "2099-12-31T00:00:00.000Z", "ativo": true } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | auth / não-ADMIN |
| 404 | `Mensalista não encontrado` |

**Exemplo de chamada**
```bash
curl -X PATCH http://localhost:5000/api/v1/mensalistas/665f.../delete -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `src/applications/routes/mensalista.route.ts` (montada em `src/applications/routes/routes.ts` como `/mensalistas`)
- Controllers: `src/applications/controllers/mensalista/{create,read,list,update,delete}/*.controller.ts`
- Usecases: `src/domain/usecases/mensalista/{create,read,list,update,delete}/*.usecase.ts`
- Serviço de domínio compartilhado: `src/domain/usecases/shared/event-scheduling-impact.service.ts` (release no cascade do delete)
- Adapters: `src/infra/adapters/mensalista/{create,read,list,update,delete,recalculate-status}/*.adapter.ts` + `src/infra/adapters/mensalista_plano/cancel-by-mensalista/*.adapter.ts`
- Schema: `src/infra/schemas/mensalista.schema.ts` (coleção `mensalistas`, soft-delete manual)
- DTO: `src/applications/dto/mensalista.dto.ts`
- Modelo: `src/domain/models/mensalista.model.ts`
- Ports: `src/domain/ports/input/mensalista.input-port.ts`, `src/domain/ports/output/mensalista-persistence.port.ts`
- Migração de legados: `src/scripts/migrate-mensalista-from-events.ts` (`npm run migrate:mensalista`)
