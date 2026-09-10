# Ranking

## Visão geral

O recurso `ranking` é a **entidade-mãe de um ranking**, exposto pelo serviço `beach-center-bff-campeonatos` sob `/api/v1/rankings`. Tem **dois modos**:

| `modo` | Significado |
|---|---|
| `PARTIDAS` | Liga com partidas marcadas — dona de [`partida`](partida.md)s (via `id_ranking`), que podem ser agendadas em `beach-center-bff-agendamentos` (`/ranking-agendamentos`, rota interna). |
| `GERAL` | Evento por período (`data_inicio`/`data_fim`) — **não** gera partida nem agendamento nenhum. Durante o período, o usuário reserva a quadra pelo fluxo comum de `agendamentos`. É só cadastro leve. |

Serviço bootstrapado na task 005. `beach-center-bff-campeonatos` é o dono do dado; o bloqueio de quadra é de `agendamentos`.

**Conceitos-chave:**

- **`status`**: `ATIVO` | `CANCELADO`. `DELETE` é soft-delete via `status='CANCELADO'`.
- **Regra `modo` × datas** (`assertRankingModoConsistency`, no usecase):
  - `GERAL` **exige** `data_inicio` **e** `data_fim`, com `data_inicio < data_fim`.
  - `PARTIDAS` **não aceita** `data_inicio`/`data_fim`.
- **`modo` é imutável** — o `PATCH` revalida `data_inicio`/`data_fim` contra o `modo` existente.
- **`agendar`** (`POST /:id/agendar`) só funciona para `modo='PARTIDAS'` e `status='ATIVO'`. Empurra `N` bloqueios de quadra para `agendamentos` de forma **atômica** (qualquer conflito rejeita o lote inteiro — sem `DRAFT`, sem prioridade).
- Cancelar o ranking (`PATCH /:id/delete`) **libera todos** os bloqueios de quadra dele em `agendamentos` (`delete-bulk` sem `ids`).

## Autenticação/autorização

**Todos os endpoints** exigem token Firebase válido **e** `user_type = ADMIN` (`authMiddleware` + `requireRole("ADMIN")`). Sem token → **401**; não-ADMIN → **403**.

## Endpoints

### POST /api/v1/rankings

Cria um ranking. Aciona `CreateRankingUsecase` (valida `modo` × datas). Nasce `status: "ATIVO"`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Sim | |
| `modalidade` | string | Sim | Texto livre |
| `modo` | string | Sim | `PARTIDAS` \| `GERAL` |
| `data_inicio` | string | Condicional | `"YYYY-MM-DD"` — **obrigatório** se `modo='GERAL'`, **proibido** se `PARTIDAS` |
| `data_fim` | string | Condicional | `"YYYY-MM-DD"` — idem; `data_inicio < data_fim` |

**Resposta de sucesso** — `201 Created`
```json
{ "message": "Ranking criado com sucesso", "data": { "id": "665a...", "nome": "Ranking Interno 2026", "modalidade": "Beach Tênis", "modo": "PARTIDAS", "status": "ATIVO" } }
```
```json
{ "message": "Ranking criado com sucesso", "data": { "id": "665a...", "nome": "Temporada de Verão", "modalidade": "Beach Tênis", "modo": "GERAL", "data_inicio": "2026-12-01", "data_fim": "2027-02-28", "status": "ATIVO" } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO — `modo` fora de `PARTIDAS`/`GERAL`, formato de data) / `Ranking 'GERAL' exige data_inicio e data_fim` / `data_inicio deve ser anterior a data_fim` / `Ranking 'PARTIDAS' não aceita data_inicio/data_fim` |
| 401 / 403 | Auth |
| 500 | `Erro interno no servidor` |

**Exemplo de chamada**
```bash
curl -X POST http://campeonatos:5003/api/v1/rankings \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{ "nome": "Ranking Interno 2026", "modalidade": "Beach Tênis", "modo": "PARTIDAS" }'
```

---

### GET /api/v1/rankings

Lista todos os rankings. Aciona `ListRankingsUsecase` (sem filtros).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Rankings listados com sucesso", "data": [ { "id": "665a...", "nome": "Ranking Interno 2026", "modalidade": "Beach Tênis", "modo": "PARTIDAS", "status": "ATIVO" } ] }
```

**Erros possíveis**: `401`, `403`, `500`.

---

### GET /api/v1/rankings/:id

Busca um ranking pelo `id`. Aciona `ReadRankingUsecase`.

**Resposta de sucesso** — `200 OK` — `{ "message": "Ranking encontrado com sucesso", "data": { ... } }`

**Erros possíveis**: `400 ID inválido`, `401`, `403`, `404 Ranking não encontrado`.

---

### PATCH /api/v1/rankings/:id

Atualiza `nome`, `modalidade`, `data_inicio` e/ou `data_fim`. Aciona `UpdateRankingUsecase`. `modo` **não** é editável — o usecase revalida as datas contra o `modo` existente.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Não | |
| `modalidade` | string | Não | |
| `data_inicio` | string | Não | `"YYYY-MM-DD"` — só faz sentido para `modo='GERAL'` |
| `data_fim` | string | Não | `"YYYY-MM-DD"` |

**Resposta de sucesso** — `200 OK` — `{ "message": "Ranking atualizado com sucesso", "data": { ... } }`

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` / `Dados inválidos` / `Ranking 'GERAL' exige data_inicio e data_fim` / `data_inicio deve ser anterior a data_fim` / `Ranking 'PARTIDAS' não aceita data_inicio/data_fim` |
| 401 / 403 | Auth |
| 404 | `Ranking não encontrado` |

---

### PATCH /api/v1/rankings/:id/delete

Cancela o ranking (soft-delete `status='CANCELADO'`). Aciona `DeleteRankingUsecase`: lê (404 senão) → `status → "CANCELADO"` → `DELETE /ranking-agendamentos/delete-bulk` (sem `ids`) em `agendamentos` (libera todos os bloqueios do ranking).

**Resposta de sucesso** — `200 OK` — `{ "message": "Ranking removido com sucesso", "data": { "id": "665a...", "status": "CANCELADO", "...": "..." } }`

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | Auth |
| 404 | `Ranking não encontrado` |
| 500 | Integração com `agendamentos` não configurada; ou erro repassado |

---

### POST /api/v1/rankings/:id/agendar

Agenda as partidas pendentes do ranking. Aciona `AgendarRankingUsecase`:

1. Lê o ranking (404 senão); precisa estar `ATIVO` (400) e `modo='PARTIDAS'` (400 `Ranking 'GERAL' não tem partidas pra agendar`).
2. Lista partidas `ATIVA` sem `id_agendamento` (400 `Nenhuma partida pendente de agendamento para este ranking` se vazio).
3. Chama `POST /ranking-agendamentos` em `agendamentos` com `quantidade` = nº de partidas pendentes. **Atômico** — se qualquer slot conflitar, `agendamentos` responde 409 e nada é persistido lá.
4. Liga cada partida ao `id` retornado, por índice.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (path) | Sim | ObjectId (24 hex) do ranking |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `quadras` | string[] | Sim | Mínimo 1 |
| `datas` | string[] | Sim | `"YYYY-MM-DD"`. Mínimo 1 |
| `hora_inicio` | string | Sim | `"HH:MM"` |
| `duracao_partida_minutos` | number | Sim | Inteiro positivo |

> Não há `cancelar_conflitos` — ranking não tem fluxo de decisão.

**Resposta de sucesso** — `201 Created` (`data` = partidas atualizadas com `id_agendamento`)
```json
{ "message": "Partidas do ranking agendadas com sucesso", "data": [ { "id": "665b...", "id_ranking": "665a...", "id_agendamento": "665f...", "lado_a": { "tipo": "SOLO", "participante": "João", "tel_participante": "..." }, "lado_b": { "tipo": "SOLO", "participante": "Ana", "tel_participante": "..." }, "status": "ATIVA" } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO) / `id_ranking inválido` / `Ranking não está ativo` / `Ranking 'GERAL' não tem partidas pra agendar` / `Nenhuma partida pendente de agendamento para este ranking` |
| 401 / 403 | Auth |
| 404 | `Ranking não encontrado` |
| 409 | `Conflito de horario com outro agendamento na mesma quadra (ranking).` — repassado de `agendamentos` |
| 500 | Integração não configurada; ou erro repassado |

**Exemplo de chamada**
```bash
curl -X POST http://campeonatos:5003/api/v1/rankings/665a.../agendar \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{ "quadras": ["665c..."], "datas": ["2026-10-18"], "hora_inicio": "20:00", "duracao_partida_minutos": 60 }'
```

## Referências

- Rota: `src/applications/routes/ranking.route.ts` (montada em `routes.ts` como `/rankings`; `authMiddleware` + `requireRole("ADMIN")` por rota)
- Middleware: `src/applications/middlewares/auth.middleware.ts`
- Controllers: `src/applications/controllers/ranking/{create,read,list,update,delete,agendar}/*.controller.ts`
- Usecases: `src/domain/usecases/ranking/{create,read,list,update,delete,agendar}/*.usecase.ts` + `shared/ranking-modo.validator.ts`
- Adapters (persistência): `src/infra/adapters/ranking/{create,read,list,update,set-status}/*.adapter.ts`
- Adapters (client → agendamentos): `src/infra/adapters/ranking-agendamento-client/{create-bulk,delete-bulk,trocar-dia}/*.adapter.ts`
- Schema: `src/infra/schemas/ranking.schema.ts` (coleção `rankings`)
- DTOs: `src/applications/dto/ranking.dto.ts`, `src/applications/dto/ranking-agendar.dto.ts`
- Modelo: `src/domain/models/ranking.model.ts`
- Ports: `src/domain/ports/input/ranking.input-port.ts`, `src/domain/ports/output/ranking-persistence.port.ts`, `src/domain/ports/output/ranking-agendamento-client.port.ts`
- Env: `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY` (opcionais)
- Serviço consumido: `beach-center-bff-agendamentos` — ver [`ranking-agendamento.md`](../beach-center-bff-agendamentos/ranking-agendamento.md)
- Recursos relacionados: [`partida.md`](partida.md), [`campeonato.md`](campeonato.md)
