# Ranking — Agendamento (bloqueio de quadra + comprovante)

## Visão geral

O recurso `ranking-agendamento` é o **bloqueio de quadra de uma partida marcada de ranking**. É exposto pelo serviço `beach-center-bff-agendamentos` sob `/api/v1/ranking-agendamentos`. Um **único router flat** mistura:

- **rotas internas** (`x-api-key`) — CRUD, chamado pelo `beach-center-bff-campeonatos` (dono do ranking/partidas), mesmo padrão de `campeonato-agendamento`;
- **uma rota pública** (`PATCH /protocolo/:numero_protocolo/comprovante`, sem token nenhum) — a pessoa que fez o agendamento anexa o comprovante de pagamento via QR code (mesmo padrão de `reserva.route.ts`).

Introduzido pela task 005. Cada registro = **1 quadra = 1 data = 1 partida**. `id_ranking` é uma referência **opaca** ao ranking dono, no MS de campeonatos.

**Diferenças em relação a `campeonato_agendamento`:**

| | `campeonato_agendamento` | `ranking_agendamento` |
|---|---|---|
| Fluxo de conflito | 2 estados (409 + `cancelar_conflitos`), pode virar `DRAFT` | **atômico** — qualquer conflito rejeita o lote inteiro (409), nada persistido |
| Prioridade sobre outras fontes | Sim (pode cancelar conflitantes) | Não — rejeita direto, como `aula_bloqueio`/`mensalista_plano` |
| Janela de bloqueio | Estende a última partida do dia até o fechamento da unidade | **Janela exata** `hora_inicio`–`hora_fim`, sem extensão |
| Pagamento | Não | `numero_protocolo` único global + comprovante (`status` `pending` → `waiting_approve`) |

Um `ranking_agendamento` não-`cancelled` e não-`expired` é a **5ª fonte** do `EventConflictService` (bloqueio pontual por data). O bloqueio vale **desde a criação** (`status: 'pending'`), independente do andamento do pagamento — não há um `CONFIRMED` separado.

**Conceitos-chave:**

- **`status`**: `pending` | `waiting_approve` | `approved` | `rejected` | `cancelled` | `expired` (tipo próprio, independente de `IReserveStatus` da reserva comum).
- **Expiração lazy (task 006a)**: um `ranking_agendamento` `pending` (**sem comprovante**) com `createdAt` anterior a `agora − 24 h` é marcado `expired` e tem o bloqueio da sua janela liberado. A varredura (`ExpirePendingRankingAgendamentosUsecase`) roda a cada `GET /api/v1/ranking-agendamentos` — não há cron. `waiting_approve` (comprovante já enviado) **nunca** expira.
- **`numero_protocolo`**: 10 dígitos, gerado com `customAlphabet('0123456789', 10)`, **único globalmente** — checado contra `reservas.number` **e** `ranking_agendamentos.numero_protocolo` antes de persistir (regenera em caso de colisão).
- **`comprovante`** (subdocumento, opcional): `{ file_name, mime_type, storage: 'google_drive' | 'database', drive_file_id?, data?, view_url?, preview_url? }`. Com as env vars `GOOGLE_DRIVE_*` configuradas → upload real no Drive (`storage: 'google_drive'`). Sem elas → fallback base64 no próprio documento (`storage: 'database'`, campo `data`).
- **`unit` é derivado** de `quadras[0]` no `POST` (todas as quadras devem pertencer à mesma unidade).
- **Alocação de slots**: mesmo `SlotAllocator` do campeonato, mas **sem** extensão da última partida (`extendLastSlotToClosing: false`).
- **`aprovar`/`rejeitar` comprovante**: **fora de escopo** da task 005 — o agendamento fica parado em `waiting_approve`.

## Autenticação/autorização

| Endpoints | Auth |
|---|---|
| `POST /`, `GET /`, `GET /auditoria`, `GET /:id`, `PATCH /delete-bulk`, `PATCH /:id`, `PATCH /:id/delete`, `PATCH /:id/trocar-dia` | Header `x-api-key` = `AGENDAMENTOS_INTERNAL_API_KEY` (`internalApiKeyMiddleware` **por rota**). Sem/errada → **401** |
| `PATCH /protocolo/:numero_protocolo/comprovante` | **Pública** — nenhum middleware. Funciona sem token |

## Endpoints

### POST /api/v1/ranking-agendamentos

Cria os agendamentos (unitário `quantidade: 1` ou em massa `N`) — **atômico**. Aciona `CreateBulkRankingAgendamentosUsecase`:

1. Deriva `unit` de `quadras` (400 se inconsistente).
2. Aloca os `quantidade` slots (janela exata; 400 `Quadras/datas insuficientes` se não couberem).
3. Para **cada** slot: se houver qualquer bloqueador `CONFIRMED` **ou** reserva ativa → **409** `RankingAgendamentoConflictError`, **nada persistido**.
4. Gera `numero_protocolo` único para cada registro, persiste com `status: 'pending'` e aplica o bloqueio (janela exata) em todos.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_ranking` | string | Sim | ObjectId (24 hex) — referência opaca ao ranking (modo `PARTIDAS`) |
| `quadras` | string[] | Sim | ObjectIds (24 hex). Mínimo 1. Mesma unidade |
| `datas` | string[] | Sim | `"YYYY-MM-DD"[]`. Mínimo 1 |
| `hora_inicio` | string | Sim | `"HH:MM"` |
| `duracao_partida_minutos` | number | Sim | Inteiro ≥ 1 |
| `modalidade` | string | Sim | Texto livre |
| `quantidade` | number | Sim | Inteiro ≥ 1 |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Agendamentos de ranking criados com sucesso",
  "data": [
    {
      "id": "665f...",
      "id_ranking": "665a...",
      "numero_protocolo": "0348172950",
      "data": "2026-10-18",
      "hora_inicio": "20:00",
      "hora_fim": "21:00",
      "court": "665c...",
      "unit": "665d...",
      "modalidade": "Beach Tênis",
      "status": "pending"
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO) / `Informe uma quantidade de partidas maior que zero` / `Quadra invalida ou sem unidade associada` / `Todas as quadras devem pertencer a mesma unidade` / `Quadras/datas insuficientes: ...` |
| 401 | `Unauthorized` |
| 409 | `Conflito na quadra <court> em <data> <hh:mm>-<hh:mm>` / `Ja existe reserva ativa na quadra ...` (`RankingAgendamentoConflictError`) |
| 500 | Erro interno |

**Exemplo de chamada**
```bash
curl -X POST http://agendamentos:5000/api/v1/ranking-agendamentos \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "id_ranking": "665a...", "quadras": ["665c..."], "datas": ["2026-10-18"], "hora_inicio": "20:00", "duracao_partida_minutos": 60, "modalidade": "Beach Tênis", "quantidade": 2 }'
```

---

### GET /api/v1/ranking-agendamentos

Lista agendamentos com filtros opcionais. Aciona `ListRankingAgendamentosUsecase`, que **antes de listar roda a varredura de expiração** (`ExpirePendingRankingAgendamentosUsecase`): todo `ranking_agendamento` `pending` com mais de 24 h de `createdAt` sem comprovante passa a `expired` e libera o bloqueio da sua janela. Lazy, sem cron.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_ranking` | string | Não | Filtra pelo ranking |
| `status` | string | Não | `pending` \| `waiting_approve` \| `approved` \| `rejected` \| `cancelled` \| `expired` |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamentos de ranking listados com sucesso", "data": [ { "id": "665f...", "numero_protocolo": "0348172950", "status": "pending", "...": "..." } ] }
```

**Erros possíveis**: `401 Unauthorized`, `500`.

---

### GET /api/v1/ranking-agendamentos/auditoria

Lista o histórico de trocas de dia. Aciona `ListRankingAgendamentoAuditoriaUsecase`. Segmento literal — registrado **antes** de `/:id`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_ranking` | string | Não | Filtra por ranking |
| `agendamento_id` | string | Não | Filtra por um agendamento específico |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Auditoria de troca de dia listada com sucesso",
  "data": [
    { "id": "6661...", "agendamento_id": "665f...", "id_ranking": "665a...", "usuario_nome": "Ana Admin", "motivo": "Reagendado a pedido do atleta", "dia_anterior": "2026-10-18", "dia_novo": "2026-10-25", "created_at": "2026-09-09T18:30:00.000Z" }
  ]
}
```

**Erros possíveis**: `401 Unauthorized`, `500`.

---

### GET /api/v1/ranking-agendamentos/:id

Busca um agendamento pelo `id`. Aciona `ReadRankingAgendamentoUsecase`.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamento de ranking encontrado com sucesso", "data": { "id": "665f...", "id_ranking": "665a...", "numero_protocolo": "0348172950", "data": "2026-10-18", "hora_inicio": "20:00", "hora_fim": "21:00", "court": "665c...", "unit": "665d...", "modalidade": "Beach Tênis", "status": "pending" } }
```

**Erros possíveis**: `400 ID inválido`, `401 Unauthorized`, `404 Agendamento de ranking não encontrado`.

---

### PATCH /api/v1/ranking-agendamentos/delete-bulk

Cancela agendamentos em massa. Aciona `DeleteBulkRankingAgendamentosUsecase`. Sem `ids` → todos do `id_ranking`; com `ids` → só os informados. Libera o `scheduling` de cada um que não estava já `cancelled`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_ranking` | string | Sim | ObjectId (24 hex) |
| `ids` | string[] | Não | ObjectIds (24 hex). Ausente = todos do ranking |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamentos de ranking removidos com sucesso", "data": [ { "id": "665f...", "status": "cancelled", "...": "..." } ] }
```

**Erros possíveis**: `400 Dados inválidos`, `401 Unauthorized`, `500`.

---

### PATCH /api/v1/ranking-agendamentos/protocolo/:numero_protocolo/comprovante

**Rota pública** (sem token). Anexa o comprovante de pagamento. Aciona `AttachProofRankingAgendamentoUsecase`:

1. Busca o agendamento por `numero_protocolo` (404 senão).
2. Se `status !== 'pending'` → **409** (`Comprovante já foi enviado para este agendamento`) — não substitui.
3. Faz upload (Google Drive se configurado, senão base64 em banco) e seta `comprovante` + `status: 'waiting_approve'` num único update.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `numero_protocolo` | string (path) | Sim | O protocolo de 10 dígitos do agendamento |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `file_name` | string | Sim | Nome do arquivo (ex.: `comprovante.pdf`) |
| `mime_type` | string | Sim | `application/pdf` \| `image/png` \| `image/jpeg` |
| `base64` | string | Sim | Conteúdo do arquivo em base64 |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Comprovante anexado com sucesso",
  "data": {
    "id": "665f...",
    "numero_protocolo": "0348172950",
    "status": "waiting_approve",
    "comprovante": {
      "file_name": "0348172950-comprovante.pdf",
      "mime_type": "application/pdf",
      "storage": "google_drive",
      "drive_file_id": "1AbC...",
      "view_url": "https://drive.google.com/file/d/1AbC.../view",
      "preview_url": "https://drive.google.com/file/d/1AbC.../preview"
    },
    "...": "..."
  }
}
```
> Sem env vars do Drive: `comprovante` vem com `"storage": "database"` e `"data": "<base64>"` (sem `drive_file_id`/`view_url`/`preview_url`).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `mime_type` fora da lista, `file_name`/`base64` ausentes |
| 404 | `Agendamento de ranking não encontrado para este protocolo` |
| 409 | `Comprovante já foi enviado para este agendamento` (`status` já ≠ `pending`) |
| 500 | Falha no upload para o Google Drive (`Falha ao enviar comprovante para o Google Drive: <status>`) |

**Exemplo de chamada**
```bash
curl -X PATCH http://agendamentos:5000/api/v1/ranking-agendamentos/protocolo/0348172950/comprovante \
  -H "Content-Type: application/json" \
  -d '{ "file_name": "comprovante.pdf", "mime_type": "application/pdf", "base64": "JVBERi0xLjQK..." }'
```

---

### PATCH /api/v1/ranking-agendamentos/:id

Update **não-estrutural** — só `modalidade`. Aciona `UpdateRankingAgendamentoUsecase`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `modalidade` | string | Não | Texto livre |

**Resposta de sucesso** — `200 OK` — `{ "message": "Agendamento de ranking atualizado com sucesso", "data": { ... } }`

**Erros possíveis**: `400 ID inválido`, `401 Unauthorized`, `404 Agendamento de ranking não encontrado`.

---

### PATCH /api/v1/ranking-agendamentos/:id/delete

Cancela um agendamento. Aciona `DeleteRankingAgendamentoUsecase`: `status='cancelled'` + libera o `scheduling` (se não estava já `cancelled`).

**Resposta de sucesso** — `200 OK` — `{ "message": "Agendamento de ranking removido com sucesso", "data": { "id": "665f...", "status": "cancelled", "...": "..." } }`

**Erros possíveis**: `400 ID inválido`, `401 Unauthorized`, `404 Agendamento de ranking não encontrado`.

---

### PATCH /api/v1/ranking-agendamentos/:id/trocar-dia

Troca **somente o dia** (`data`). Aciona `ChangeDayRankingAgendamentoUsecase`. Diferente do campeonato: conflito no novo dia é **rejeição simples 409** (`RankingAgendamentoConflictError`), sem fluxo de decisão / `cancelar_conflitos`.

1. Valida `motivo` e `id`; lê o agendamento (404 senão); não pode estar `cancelled` (400).
2. `dia_novo` diferente do atual e não passado (400).
3. Checa conflito no novo slot (bloqueadores + reservas ativas, excluindo o próprio) → **409** se houver.
4. Libera o slot antigo, atualiza `data`, aplica bloqueio (janela exata) no novo, grava auditoria.

> `usuario_nome` vem **no corpo** (rota interna).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dia_novo` | string | Sim | `"YYYY-MM-DD"`, diferente do atual, não passado |
| `motivo` | string | Sim | Texto livre — gravado na auditoria |
| `usuario_nome` | string | Sim | Nome do admin (para a auditoria) |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Dia do agendamento de ranking trocado com sucesso", "data": { "id": "665f...", "data": "2026-10-25", "numero_protocolo": "0348172950", "status": "pending", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` / `Motivo é obrigatório para trocar o dia` / `ID inválido` / `Não é possível trocar o dia de um agendamento cancelado` / `O novo dia deve ser diferente do dia atual` / `Não é possível trocar para uma data passada` |
| 401 | `Unauthorized` |
| 404 | `Agendamento de ranking não encontrado` |
| 409 | `Conflito na quadra ...` / `Ja existe reserva ativa na quadra ...` (`RankingAgendamentoConflictError`) |

## Referências

- Rota: `src/applications/routes/ranking-agendamento.route.ts` (montada em `routes.ts` como `/ranking-agendamentos`; `internalApiKeyMiddleware` por rota, exceto a pública de comprovante)
- Middleware: `src/applications/middlewares/internal-api-key.middleware.ts`
- Controllers: `src/applications/controllers/ranking_agendamento/{create-bulk,read,list,list-auditoria,update,delete,change-day,attach-proof}/*.controller.ts`
- Usecases: `src/domain/usecases/ranking-agendamento/{create-bulk,read,list,update,delete,change-day,list-auditoria,attach-proof}/*.usecase.ts` + `shared/{ranking-agendamento-conflict.error,expire-pending-ranking-agendamentos.usecase}.ts`
- Serviços de domínio compartilhados: `src/domain/usecases/shared/{slot-allocator,event-conflict.service,event-scheduling-impact.service}.ts`
- Adapters: `src/infra/adapters/ranking_agendamento/{create-many,read,find-by-protocol,list,list-pending-older-than,update,set-status,change-data,set-comprovante,delete-bulk,find-confirmed-by-court-unit}/*.adapter.ts` + `src/infra/adapters/ranking_agendamento_auditoria/{create,list}/*.adapter.ts` + `src/infra/adapters/protocol/is-protocol-number-taken/*.adapter.ts` + `src/infra/adapters/file-storage/google-drive-file-storage.adapter.ts`
- Schemas: `src/infra/schemas/ranking-agendamento.schema.ts` (coleção `ranking_agendamentos`, índice único `numero_protocolo`), `src/infra/schemas/ranking-agendamento-auditoria.schema.ts`
- DTOs: `src/applications/dto/ranking-agendamento-bulk.dto.ts`, `src/applications/dto/ranking-agendamento.dto.ts`, `src/applications/dto/ranking-agendamento-comprovante.dto.ts`
- Modelos: `src/domain/models/ranking-agendamento.model.ts`, `src/domain/models/ranking-agendamento-auditoria.model.ts`
- Ports: `src/domain/ports/input/ranking-agendamento.input-port.ts`, `src/domain/ports/output/ranking-agendamento-persistence.port.ts`, `src/domain/ports/output/ranking-agendamento-shared.port.ts`, `src/domain/ports/output/ranking-agendamento-auditoria-persistence.port.ts`, `src/domain/ports/output/protocol-uniqueness.port.ts`, `src/domain/ports/output/file-storage.port.ts`
- Env: `GOOGLE_DRIVE_CLIENT_EMAIL`, `GOOGLE_DRIVE_PRIVATE_KEY`, `GOOGLE_DRIVE_FOLDER_ID` (opcionais — `src/config/env.ts`)
- Consumidor (outro serviço): `beach-center-bff-campeonatos` — ver [`ranking.md`](../beach-center-bff-campeonatos/ranking.md) e [`partida.md`](../beach-center-bff-campeonatos/partida.md)
- Recurso irmão: [`campeonato-agendamento.md`](campeonato-agendamento.md)
