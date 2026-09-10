# Campeonato

## Visão geral

O recurso `campeonato` é a **entidade-mãe de um campeonato** (nome, modalidade), exposto pelo serviço `beach-center-bff-campeonatos` sob `/api/v1/campeonatos`. Este serviço é o **dono do dado de negócio** de campeonato/ranking/partida; o bloqueio de quadra das partidas é responsabilidade de `beach-center-bff-agendamentos` (`/campeonato-agendamentos`, rota interna) — este serviço **empurra** o bloqueio para lá via `x-api-key`, nunca o contrário.

Serviço bootstrapado na task 005 (antes era um repo vazio). Segue Arquitetura Hexagonal, mesmo template estrutural de `beach-center-bff-aulas`.

**Conceitos-chave:**

- **`status`**: `ATIVO` | `CANCELADO`. `DELETE` é soft-delete via `status='CANCELADO'`.
- **`modalidade`**: string livre.
- As **partidas** de um campeonato ficam no recurso [`partida`](partida.md) (`id_campeonato`).
- **`agendar`** (`POST /:id/agendar`) pega as partidas `ATIVA` sem `id_agendamento` e empurra `N` bloqueios de quadra para `agendamentos`, ligando cada partida ao `id_agendamento` retornado por índice. É **idempotente** — repetir a chamada só agenda o que ainda falta.
- Cancelar o campeonato (`PATCH /:id/delete`) **libera todos** os bloqueios de quadra dele em `agendamentos` (`delete-bulk` sem `ids`).

## Autenticação/autorização

**Todos os endpoints** exigem token Firebase válido (`Authorization: Bearer <idToken>`) **e** `user_type = ADMIN` (`authMiddleware` + `requireRole("ADMIN")`).

- Sem token / token inválido → **401** (`Token invalido ou expirado`).
- Token válido mas não-ADMIN → **403** (`Acesso negado`).

## Endpoints

### POST /api/v1/campeonatos

Cria um campeonato. Aciona `CreateCampeonatoUsecase`. Nasce com `status: "ATIVO"`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Sim | Nome do campeonato |
| `modalidade` | string | Sim | Texto livre (ex.: `Beach Tênis`) |

**Resposta de sucesso** — `201 Created`
```json
{ "message": "Campeonato criado com sucesso", "data": { "id": "665a...", "nome": "Aberto de Verão", "modalidade": "Beach Tênis", "status": "ATIVO" } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `nome`/`modalidade` ausentes |
| 401 | Token ausente/inválido |
| 403 | Usuário não é ADMIN |
| 500 | `Erro interno no servidor` |

**Exemplo de chamada**
```bash
curl -X POST http://campeonatos:5003/api/v1/campeonatos \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{ "nome": "Aberto de Verão", "modalidade": "Beach Tênis" }'
```

---

### GET /api/v1/campeonatos

Lista todos os campeonatos. Aciona `ListCampeonatosUsecase` (sem filtros).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Campeonatos listados com sucesso", "data": [ { "id": "665a...", "nome": "Aberto de Verão", "modalidade": "Beach Tênis", "status": "ATIVO" } ] }
```

**Erros possíveis**: `401`, `403`, `500`.

---

### GET /api/v1/campeonatos/:id

Busca um campeonato pelo `id`. Aciona `ReadCampeonatoUsecase` (valida `id`; 404 senão).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Campeonato encontrado com sucesso", "data": { "id": "665a...", "nome": "Aberto de Verão", "modalidade": "Beach Tênis", "status": "ATIVO" } }
```

**Erros possíveis**: `400 ID inválido`, `401`, `403`, `404 Campeonato não encontrado`.

---

### PATCH /api/v1/campeonatos/:id

Atualiza `nome` e/ou `modalidade`. Aciona `UpdateCampeonatoUsecase`. `status` não muda por esta rota.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Não | |
| `modalidade` | string | Não | |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Campeonato atualizado com sucesso", "data": { "id": "665a...", "nome": "Aberto de Verão 2026", "modalidade": "Beach Tênis", "status": "ATIVO" } }
```

**Erros possíveis**: `400 ID inválido`, `401`, `403`, `404 Campeonato não encontrado`.

---

### PATCH /api/v1/campeonatos/:id/delete

Cancela o campeonato (soft-delete `status='CANCELADO'`). Aciona `DeleteCampeonatoUsecase`:

1. Lê o campeonato (404 senão).
2. `status → "CANCELADO"`.
3. Chama `DELETE /campeonato-agendamentos/delete-bulk` (sem `ids`) em `agendamentos` — libera **todos** os bloqueios de quadra do campeonato.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Campeonato removido com sucesso", "data": { "id": "665a...", "nome": "Aberto de Verão", "modalidade": "Beach Tênis", "status": "CANCELADO" } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | Auth |
| 404 | `Campeonato não encontrado` |
| 500 | `Integração com agendamentos não configurada (AGENDAMENTOS_API_URL / AGENDAMENTOS_INTERNAL_API_KEY).` — env de integração ausente; ou erro repassado por `agendamentos` |

---

### POST /api/v1/campeonatos/:id/agendar

Agenda as partidas pendentes do campeonato — empurra o bloqueio de quadra para `agendamentos`. Aciona `AgendarCampeonatoUsecase`:

1. Lê o campeonato (404 senão); precisa estar `ATIVO` (400 `Campeonato não está ativo`).
2. Lista as partidas `ATIVA` **sem** `id_agendamento` (400 `Nenhuma partida pendente de agendamento para este campeonato` se não houver).
3. Chama `POST /campeonato-agendamentos` em `agendamentos` com `quantidade = nº de partidas pendentes` + `modalidade` do campeonato + as `quadras`/`datas`/`hora_inicio`/`duracao_partida_minutos` recebidas (+ `cancelar_conflitos` se veio).
4. Liga cada partida pendente ao `id` do agendamento retornado, **na mesma ordem** (idempotente).

**Fluxo de conflito ponta a ponta:** se `agendamentos` responder **409** (`CAMPEONATO_AGENDAMENTO_CONFLICT`), este endpoint repassa `409` com a mesma lista `conflitos[]`. O ADMIN resubmete com `cancelar_conflitos: true` (cancela os conflitantes e confirma) ou `false` (cria os bloqueios como `DRAFT` do lado de `agendamentos`).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do campeonato |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `quadras` | string[] | Sim | Quadras candidatas. Mínimo 1 |
| `datas` | string[] | Sim | Datas candidatas `"YYYY-MM-DD"`. Mínimo 1 |
| `hora_inicio` | string | Sim | `"HH:MM"` |
| `duracao_partida_minutos` | number | Sim | Inteiro positivo |
| `cancelar_conflitos` | boolean | Não | Repassado ao 2º submit em `agendamentos` |

**Resposta de sucesso** — `201 Created` (`data` = partidas atualizadas, já com `id_agendamento`)
```json
{
  "message": "Partidas do campeonato agendadas com sucesso",
  "data": [
    {
      "id": "665b...",
      "id_campeonato": "665a...",
      "id_agendamento": "665f...",
      "lado_a": { "tipo": "DUPLA", "participante_a": "João", "tel_participante_a": "...", "participante_b": "Pedro", "tel_participante_b": "..." },
      "lado_b": { "tipo": "DUPLA", "participante_a": "Ana", "tel_participante_a": "...", "participante_b": "Bia", "tel_participante_b": "..." },
      "status": "ATIVA"
    }
  ]
}
```

**Resposta de conflito** — `409 Conflict`
```json
{
  "message": "Existem conflitos nos slots do campeonato. Confirme 'cancelar_conflitos' para prosseguir.",
  "code": "CAMPEONATO_AGENDAMENTO_CONFLICT",
  "conflitos": [ { "candidato_index": 0, "tipo": "BLOQUEADOR", "source": "AULA", "id": "665e...", "data": "2026-10-18", "hora_inicio": "14:00", "hora_fim": "15:00", "court": "665c..." } ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO) / `id_campeonato inválido` / `Campeonato não está ativo` / `Nenhuma partida pendente de agendamento para este campeonato` |
| 401 / 403 | Auth |
| 404 | `Campeonato não encontrado` |
| 409 | `CAMPEONATO_AGENDAMENTO_CONFLICT` — repassado de `agendamentos` |
| 500 | Integração não configurada; ou erro repassado por `agendamentos` |

**Exemplo de chamada**
```bash
curl -X POST http://campeonatos:5003/api/v1/campeonatos/665a.../agendar \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{ "quadras": ["665c..."], "datas": ["2026-10-18"], "hora_inicio": "14:00", "duracao_partida_minutos": 60 }'
```

## Referências

- Rota: `src/applications/routes/campeonato.route.ts` (montada em `routes.ts` como `/campeonatos`; `authMiddleware` + `requireRole("ADMIN")` por rota)
- Middleware: `src/applications/middlewares/auth.middleware.ts`
- Controllers: `src/applications/controllers/campeonato/{create,read,list,update,delete,agendar}/*.controller.ts`
- Usecases: `src/domain/usecases/campeonato/{create,read,list,update,delete,agendar}/*.usecase.ts`
- Adapters (persistência): `src/infra/adapters/campeonato/{create,read,list,update,set-status}/*.adapter.ts`
- Adapters (client → agendamentos): `src/infra/adapters/campeonato-agendamento-client/{create-bulk,delete-bulk,trocar-dia}/*.adapter.ts`
- Schema: `src/infra/schemas/campeonato.schema.ts` (coleção `campeonatos`)
- DTOs: `src/applications/dto/campeonato.dto.ts`, `src/applications/dto/campeonato-agendar.dto.ts`
- Modelo: `src/domain/models/campeonato.model.ts`
- Ports: `src/domain/ports/input/campeonato.input-port.ts`, `src/domain/ports/output/campeonato-persistence.port.ts`, `src/domain/ports/output/campeonato-agendamento-client.port.ts`
- Erro compartilhado: `src/domain/usecases/shared/campeonato-agendamento-conflict.error.ts`
- Env: `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY` (opcionais — `src/config/env.ts`)
- Serviço consumido: `beach-center-bff-agendamentos` — ver [`campeonato-agendamento.md`](../beach-center-bff-agendamentos/campeonato-agendamento.md)
- Recursos relacionados: [`partida.md`](partida.md), [`ranking.md`](ranking.md)
