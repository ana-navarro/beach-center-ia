# Partida

## Visão geral

O recurso `partida` são os **participantes e o confronto de uma partida** (chaveamento), exposto pelo serviço `beach-center-bff-campeonatos` sob `/api/v1/partidas`. Uma partida pertence a **exatamente um** campeonato **ou** um ranking (nunca os dois, nunca nenhum — XOR).

O bloqueio de quadra da partida (quando ela é agendada) vive em `beach-center-bff-agendamentos` (`campeonato_agendamento`/`ranking_agendamento`). Aqui a partida guarda só `id_agendamento` — uma **referência opaca** ao bloqueio, preenchida pelos endpoints `agendar` de [`campeonato`](campeonato.md)/[`ranking`](ranking.md) e usada por `PATCH /partidas/:id/trocar-dia`.

**Composição dos lados (`lado_a` / `lado_b`)** — cada lado tem um discriminante `tipo` explícito:

| `tipo` | Campos |
|---|---|
| `DUPLA` | `participante_a`, `tel_participante_a`, `participante_b`, `tel_participante_b` |
| `SOLO` | `participante`, `tel_participante` |
| `EQUIPE` | `participantes: string[]` (mínimo 2), `tel_capitao` |

**Regra de composição:** `lado_a.tipo === lado_b.tipo` — dupla só joga contra dupla, solo contra solo, equipe contra equipe. Validada no DTO (`yup`) **e** no usecase (`assertPartidaComposicaoValida`, defesa em profundidade).

**Conceitos-chave:**

- **`status`**: `ATIVA` | `CANCELADA`. `DELETE` é soft-delete via `status='CANCELADA'`.
- Ao **criar**, o dono (`id_campeonato` ou `id_ranking`) precisa existir e estar `ATIVO`. Se ranking, precisa estar no `modo='PARTIDAS'`.
- O dono (`id_campeonato`/`id_ranking`) é **imutável** — para trocar, cancele e recrie.
- Cancelar uma partida já agendada libera **só** o bloqueio dela em `agendamentos` (`delete-bulk` com `ids: [id_agendamento]`) — não afeta as outras partidas do mesmo campeonato/ranking.

## Autenticação/autorização

**Todos os endpoints** exigem token Firebase válido **e** `user_type = ADMIN` (`authMiddleware` + `requireRole("ADMIN")`). Sem token → **401**; não-ADMIN → **403**.

## Endpoints

### POST /api/v1/partidas

Cria uma partida. Aciona `CreatePartidaUsecase`:

1. Valida XOR de `id_campeonato`/`id_ranking` (400 `Informe exatamente um de 'id_campeonato' ou 'id_ranking'`).
2. Lê o dono (404 senão); precisa estar `ATIVO` (400). Se ranking, `modo='PARTIDAS'` (400 `Ranking 'GERAL' não tem partidas marcadas`).
3. Valida a composição (`lado_a.tipo === lado_b.tipo`).
4. Persiste com `status: "ATIVA"`, sem `id_agendamento`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Condicional | ObjectId (24 hex) — **exatamente um** de `id_campeonato`/`id_ranking` |
| `id_ranking` | string | Condicional | ObjectId (24 hex) |
| `lado_a` | objeto | Sim | Participante (`DUPLA` \| `SOLO` \| `EQUIPE`) — ver tabela de composição |
| `lado_b` | objeto | Sim | Participante — mesmo `tipo` que `lado_a` |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Partida criada com sucesso",
  "data": {
    "id": "665b...",
    "id_campeonato": "665a...",
    "lado_a": { "tipo": "DUPLA", "participante_a": "João", "tel_participante_a": "5548999...", "participante_b": "Pedro", "tel_participante_b": "5548988..." },
    "lado_b": { "tipo": "DUPLA", "participante_a": "Ana", "tel_participante_a": "5548977...", "participante_b": "Bia", "tel_participante_b": "5548966..." },
    "status": "ATIVA"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO — falta `tipo`/campos do lado, `EQUIPE` com < 2 participantes) / `Composicao de partida invalida: os dois lados devem ser do mesmo tipo (DUPLA, SOLO ou EQUIPE)` / `Informe exatamente um de 'id_campeonato' ou 'id_ranking'` / `id_campeonato inválido` / `id_ranking inválido` / `Campeonato não está ativo` / `Ranking não está ativo` / `Ranking 'GERAL' não tem partidas marcadas` |
| 401 / 403 | Auth |
| 404 | `Campeonato não encontrado` / `Ranking não encontrado` |
| 500 | `Erro interno no servidor` |

**Exemplo de chamada**
```bash
curl -X POST http://campeonatos:5003/api/v1/partidas \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "id_campeonato": "665a...",
    "lado_a": { "tipo": "DUPLA", "participante_a": "João", "tel_participante_a": "5548999...", "participante_b": "Pedro", "tel_participante_b": "5548988..." },
    "lado_b": { "tipo": "DUPLA", "participante_a": "Ana", "tel_participante_a": "5548977...", "participante_b": "Bia", "tel_participante_b": "5548966..." }
  }'
```

---

### GET /api/v1/partidas

Lista partidas com filtros opcionais via query. Aciona `ListPartidasUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Não | Filtra pelo campeonato |
| `id_ranking` | string | Não | Filtra pelo ranking |
| `status` | string | Não | `ATIVA` \| `CANCELADA` |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Partidas listadas com sucesso", "data": [ { "id": "665b...", "id_campeonato": "665a...", "id_agendamento": "665f...", "lado_a": { "...": "..." }, "lado_b": { "...": "..." }, "status": "ATIVA" } ] }
```

**Erros possíveis**: `401`, `403`, `500`.

---

### GET /api/v1/partidas/:id

Busca uma partida pelo `id`. Aciona `ReadPartidaUsecase`.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Partida encontrada com sucesso", "data": { "id": "665b...", "id_campeonato": "665a...", "id_agendamento": "665f...", "lado_a": { "...": "..." }, "lado_b": { "...": "..." }, "status": "ATIVA" } }
```

**Erros possíveis**: `400 ID inválido`, `401`, `403`, `404 Partida não encontrada`.

---

### PATCH /api/v1/partidas/:id

Atualiza **só** `lado_a` e/ou `lado_b`. Aciona `UpdatePartidaUsecase`. Revalida a composição (contra o lado enviado ou o existente). O dono não muda.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `lado_a` | objeto | Não | Participante — se enviado, valida o shape do `tipo` |
| `lado_b` | objeto | Não | Participante |

**Resposta de sucesso** — `200 OK` — `{ "message": "Partida atualizada com sucesso", "data": { ... } }`

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` / `Dados inválidos` / `Composicao de partida invalida: <tipo> nao pode jogar contra <tipo>` |
| 401 / 403 | Auth |
| 404 | `Partida não encontrada` |

---

### PATCH /api/v1/partidas/:id/delete

Cancela a partida (soft-delete `status='CANCELADA'`). Aciona `DeletePartidaUsecase`:

1. Lê a partida (404 senão); `status → "CANCELADA"`.
2. Se tinha `id_agendamento`: chama `DELETE /campeonato-agendamentos/delete-bulk` (ou `/ranking-agendamentos/delete-bulk`) com `ids: [id_agendamento]` — libera **só** o bloqueio dessa partida.

**Resposta de sucesso** — `200 OK` — `{ "message": "Partida removida com sucesso", "data": { "id": "665b...", "status": "CANCELADA", "...": "..." } }`

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 / 403 | Auth |
| 404 | `Partida não encontrada` |
| 500 | Integração com `agendamentos` não configurada; ou erro repassado |

---

### PATCH /api/v1/partidas/:id/trocar-dia

Troca o dia do **bloqueio de quadra** da partida (a partida em si não guarda data/hora). Aciona `TrocarDiaPartidaUsecase`:

1. Lê a partida (404 senão); precisa estar `ATIVA` (400) e ter `id_agendamento` (400 `Partida ainda não foi agendada`).
2. Despacha para `PATCH /campeonato-agendamentos/:id_agendamento/trocar-dia` ou `PATCH /ranking-agendamentos/:id_agendamento/trocar-dia` em `agendamentos`, conforme o dono.
3. `usuario_nome` é preenchido com `req.databaseUser.name` (ADMIN autenticado neste serviço) e enviado no corpo para `agendamentos` gravar a auditoria.

**Fluxo de conflito (só campeonato):** se `agendamentos` responder `409` (`CAMPEONATO_AGENDAMENTO_CONFLICT`), este endpoint repassa `409` com a lista `conflitos[]`. Resubmeter com `cancelar_conflitos: true` cancela os conflitantes e move. Para ranking, conflito é `409` simples (sem `cancelar_conflitos`).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dia_novo` | string | Sim | `"YYYY-MM-DD"` |
| `motivo` | string | Sim | Texto livre — gravado na auditoria em `agendamentos` |
| `cancelar_conflitos` | boolean | Não | Só relevante para partida de campeonato |

**Resposta de sucesso** — `200 OK` (`data` = o bloqueio atualizado, retornado por `agendamentos`)
```json
{ "message": "Dia da partida trocado com sucesso", "data": { "id": "665f...", "data": "2026-10-25", "hora_inicio": "14:00", "hora_fim": "15:00", "court": "665c...", "unit": "665d..." } }
```

**Resposta de conflito** — `409` (`{ "message": "...", "code": "CAMPEONATO_AGENDAMENTO_CONFLICT", "conflitos": [ ... ] }`)

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO — `dia_novo` fora de formato, `motivo` ausente) / `ID inválido` / `Só é possível trocar o dia de uma partida ATIVA` / `Partida ainda não foi agendada` / `Partida sem campeonato ou ranking associado` |
| 401 / 403 | Auth |
| 404 | `Partida não encontrada` |
| 409 | `CAMPEONATO_AGENDAMENTO_CONFLICT` (campeonato) / `Conflito de horario com outro agendamento na mesma quadra (ranking).` (ranking) — repassado de `agendamentos` |
| 500 | Integração não configurada; ou erro repassado (ex.: `Não é possível trocar para uma data passada` vindo de `agendamentos` chega como erro de domínio) |

**Exemplo de chamada**
```bash
curl -X PATCH http://campeonatos:5003/api/v1/partidas/665b.../trocar-dia \
  -H "Authorization: Bearer $ID_TOKEN" -H "Content-Type: application/json" \
  -d '{ "dia_novo": "2026-10-25", "motivo": "Chuva prevista", "cancelar_conflitos": true }'
```

## Referências

- Rota: `src/applications/routes/partida.route.ts` (montada em `routes.ts` como `/partidas`; `authMiddleware` + `requireRole("ADMIN")` por rota)
- Middleware: `src/applications/middlewares/auth.middleware.ts`
- Controllers: `src/applications/controllers/partida/{create,read,list,update,delete,trocar-dia}/*.controller.ts`
- Usecases: `src/domain/usecases/partida/{create,read,list,update,delete,trocar-dia}/*.usecase.ts` + `src/domain/usecases/shared/partida-composicao.validator.ts`
- Adapters (persistência): `src/infra/adapters/partida/{create,read,list,update,set-status,set-agendamento}/*.adapter.ts`
- Adapters (client → agendamentos): `src/infra/adapters/campeonato-agendamento-client/{delete-bulk,trocar-dia}/*.adapter.ts`, `src/infra/adapters/ranking-agendamento-client/{delete-bulk,trocar-dia}/*.adapter.ts`
- Schema: `src/infra/schemas/partida.schema.ts` (coleção `partidas`; `lado_a`/`lado_b` como `Mixed`)
- DTOs: `src/applications/dto/partida.dto.ts`, `src/applications/dto/partida-trocar-dia.dto.ts`
- Modelo: `src/domain/models/partida.model.ts`
- Ports: `src/domain/ports/input/partida.input-port.ts`, `src/domain/ports/output/partida-persistence.port.ts`, `src/domain/ports/output/campeonato-agendamento-client.port.ts`, `src/domain/ports/output/ranking-agendamento-client.port.ts`
- Erro compartilhado: `src/domain/usecases/shared/campeonato-agendamento-conflict.error.ts`
- Serviço consumido: `beach-center-bff-agendamentos` — ver [`campeonato-agendamento.md`](../beach-center-bff-agendamentos/campeonato-agendamento.md) e [`ranking-agendamento.md`](../beach-center-bff-agendamentos/ranking-agendamento.md)
- Recursos relacionados: [`campeonato.md`](campeonato.md), [`ranking.md`](ranking.md)
