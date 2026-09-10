# Campeonato — Agendamento (bloqueio de quadra, rota interna)

## Visão geral

O recurso `campeonato-agendamento` é o **bloqueio de quadra de uma partida de campeonato**. É exposto pelo serviço `beach-center-bff-agendamentos` sob `/api/v1/campeonato-agendamentos`, como **rota interna serviço-a-serviço** — o dono do campeonato (nome, modalidade, partidas, chaveamento) é o `beach-center-bff-campeonatos`, que delega o bloqueio de quadra a `agendamentos` via HTTP interno (mesmo padrão de `aula-bloqueio` ← `beach-center-bff-aulas`). `agendamentos` **nunca** chama o MS de campeonatos de volta.

Introduzido pela task 005 (que tirou "campeonato" da tabela genérica `eventos_agendados`). Cada registro = **1 quadra = 1 data = 1 partida**. `id_campeonato` é uma referência **opaca** ao campeonato dono, no outro serviço — não há entidade `campeonato` local.

Um `campeonato_agendamento` `CONFIRMED` é a **4ª fonte** do `EventConflictService` (kernel de conflito unificado), junto com `eventos_agendados` (`OUTRO`), `mensalista_planos`, `aula_bloqueios` e `ranking_agendamentos`. É um bloqueio **pontual por data** (`specific_date`), não recorrente por dia-da-semana.

**Conceitos-chave:**

- **`status`**: `DRAFT` | `CONFIRMED` | `CANCELLED`.
  - `CONFIRMED` — bloqueia a quadra (`applyEventToSchedulings`) e entra no `EventConflictService`.
  - `DRAFT` — **não bloqueia nada**; existe só como resultado do fluxo de conflito (ver `POST` abaixo) quando o chamador opta por **não** cancelar os conflitos detectados. Serve de anotação até virar `CONFIRMED` (nova tentativa com `cancelar_conflitos: true`) ou trocar de dia.
  - `CANCELLED` — cancelado; libera a quadra se estava `CONFIRMED`.
- **`unit` é derivado** de `quadras[0]` no `POST` (e validado contra as demais — todas devem pertencer à mesma unidade). Não é campo de entrada.
- **Alocação de slots (`SlotAllocator`, determinística):** dado `quadras`, `datas`, `hora_inicio`, `duracao_partida_minutos` e `quantidade`, o serviço gera slots sequenciais por `data` (ordem recebida) × `quadra` (ordem recebida), a partir de `hora_inicio`, parando quando o próximo `hora_fim` ultrapassaria o **fechamento da unidade** (`22:00` para a unidade `6a440a931094fad2f585011b`, `23:00` para as demais). Os `quantidade` primeiros slots são usados. O chamador (MS de campeonatos) casa os `N` registros retornados **por índice** com as suas próprias partidas.
- **Regra de bloqueio assimétrica (campeonato):** a **última** partida de cada quadra/dia tem `hora_fim` **estendido até o fechamento da unidade** (jogo tardio "estoura" o previsto). As demais mantêm o fim exato da duração. (O `ranking_agendamento` NÃO estende — ver [`ranking-agendamento.md`](ranking-agendamento.md).)
- **`hora_inicio`/`hora_fim`** são strings `"HH:MM"`; **`data`** é `"YYYY-MM-DD"`.
- **Auditoria de troca de dia:** cada `trocar-dia` bem-sucedido grava um registro em `campeonato_agendamento_auditorias` (`usuario_nome`, `motivo`, `dia_anterior`, `dia_novo`, `created_at`).

## Autenticação/autorização

**Não usa Firebase.** Todos os endpoints exigem o header `x-api-key` igual a `AGENDAMENTOS_INTERNAL_API_KEY` (`internalApiKeyMiddleware`, aplicado com `router.use(...)` para a rota inteira). Sem a chave (ou com chave errada) → **401 `Unauthorized`**.

## Endpoints

### POST /api/v1/campeonato-agendamentos

Cria os agendamentos (unitário com `quantidade: 1` ou em massa com `N`) — mesmo contrato. Aciona `CreateBulkCampeonatoAgendamentosUsecase`.

**Fluxo de conflito em 2 estados (espelha `CloseDateUsecase` — não há endpoint separado de "dry-run"):**

1. Deriva `unit` de `quadras` (400 se quadras de unidades diferentes ou quadra sem unidade).
2. Aloca os `quantidade` slots (400 `Quadras/datas insuficientes: N partida(s) para M slot(s) disponivel(eis)` se não couberem).
3. Detecta conflitos de cada slot: **bloqueadores** (`EventConflictService` — 5 fontes `CONFIRMED`) + **reservas ativas** (`scheduling`s com reserva ativa).
4. Decisão conforme `cancelar_conflitos`:
   - **ausente** + há conflito → **409** `CampeonatoAgendamentoConflictError` com a lista `conflitos[]`, **nada persistido**. O chamador resubmete a **mesma** chamada com a decisão.
   - `true` → cancela/libera cada fonte conflitante (dispatch por `source`/`tipo`, reaproveitando o usecase de cancelamento de cada domínio) e persiste **todos** os slots como `CONFIRMED`, aplicando o bloqueio.
   - `false` → **nada é cancelado**; persiste todos os slots como `DRAFT` (sem bloquear a quadra).
   - sem conflito → persiste `CONFIRMED` direto (independente de `cancelar_conflitos`).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Sim | ObjectId (24 hex) — referência opaca ao campeonato no MS de campeonatos |
| `quadras` | string[] | Sim | ObjectIds (24 hex) das quadras candidatas. Mínimo 1. Todas na mesma unidade |
| `datas` | string[] | Sim | Datas candidatas `"YYYY-MM-DD"`. Mínimo 1 |
| `hora_inicio` | string | Sim | `"HH:MM"` — início do 1º slot de cada quadra/dia |
| `duracao_partida_minutos` | number | Sim | Inteiro ≥ 1 |
| `modalidade` | string | Sim | Texto livre |
| `quantidade` | number | Sim | Inteiro ≥ 1 — quantos slots (partidas) alocar |
| `cancelar_conflitos` | boolean | Não | Ausente = lançar 409 se houver conflito; `true` = cancelar e confirmar; `false` = criar como `DRAFT` |

**Resposta de sucesso** — `201 Created` (`data` na ordem determinística do `SlotAllocator`)
```json
{
  "message": "Agendamentos de campeonato criados com sucesso",
  "data": [
    {
      "id": "665f...",
      "id_campeonato": "665a...",
      "data": "2026-10-18",
      "hora_inicio": "14:00",
      "hora_fim": "15:00",
      "court": "665c...",
      "unit": "665d...",
      "modalidade": "Beach Tênis",
      "status": "CONFIRMED"
    }
  ]
}
```

**Resposta de conflito** — `409 Conflict`
```json
{
  "message": "Existem conflitos nos slots do campeonato. Confirme 'cancelar_conflitos' para prosseguir.",
  "code": "CAMPEONATO_AGENDAMENTO_CONFLICT",
  "conflitos": [
    {
      "candidato_index": 1,
      "data": "2026-10-18",
      "hora_inicio": "15:00",
      "hora_fim": "16:00",
      "court": "665c...",
      "tipo": "BLOQUEADOR",
      "source": "MENSALISTA",
      "id": "665e..."
    }
  ]
}
```
> `tipo`: `BLOQUEADOR` (com `source`: `EVENT` | `MENSALISTA` | `AULA` | `CAMPEONATO` | `RANKING`) ou `RESERVA` (sem `source`; `id` = id da reserva).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO) / `Informe uma quantidade de partidas maior que zero` / `Quadra invalida ou sem unidade associada` / `Todas as quadras devem pertencer a mesma unidade` / `Quadras/datas insuficientes: N partida(s) para M slot(s) disponivel(eis)` |
| 401 | `Unauthorized` — `x-api-key` ausente/errada |
| 409 | `CampeonatoAgendamentoConflictError` — há conflito e `cancelar_conflitos` não foi informado |
| 500 | Erro interno (`Erro interno no servidor`) |

**Exemplo de chamada**
```bash
# 1ª tentativa
curl -X POST http://agendamentos:5000/api/v1/campeonato-agendamentos \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "id_campeonato": "665a...", "quadras": ["665c..."], "datas": ["2026-10-18"], "hora_inicio": "14:00", "duracao_partida_minutos": 60, "modalidade": "Beach Tênis", "quantidade": 3 }'

# resubmissão após 409 (cancelar os conflitantes e confirmar)
curl -X POST http://agendamentos:5000/api/v1/campeonato-agendamentos \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "id_campeonato": "665a...", "quadras": ["665c..."], "datas": ["2026-10-18"], "hora_inicio": "14:00", "duracao_partida_minutos": 60, "modalidade": "Beach Tênis", "quantidade": 3, "cancelar_conflitos": true }'
```

---

### GET /api/v1/campeonato-agendamentos

Lista agendamentos com filtros opcionais via query. Aciona `ListCampeonatoAgendamentosUsecase`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Não | Filtra pelo campeonato |
| `status` | string | Não | `DRAFT` \| `CONFIRMED` \| `CANCELLED` |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamentos de campeonato listados com sucesso", "data": [ { "id": "665f...", "id_campeonato": "665a...", "data": "2026-10-18", "hora_inicio": "14:00", "hora_fim": "15:00", "court": "665c...", "unit": "665d...", "modalidade": "Beach Tênis", "status": "CONFIRMED" } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | `Unauthorized` |
| 500 | Erro interno |

---

### GET /api/v1/campeonato-agendamentos/auditoria

Lista o histórico de trocas de dia. Aciona `ListCampeonatoAgendamentoAuditoriaUsecase`. Segmento literal — registrado **antes** de `/:id`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Não | Filtra por campeonato (denormalizado na auditoria) |
| `agendamento_id` | string | Não | Filtra por um agendamento específico |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Auditoria de troca de dia listada com sucesso",
  "data": [
    {
      "id": "6661...",
      "agendamento_id": "665f...",
      "id_campeonato": "665a...",
      "usuario_nome": "Ana Admin",
      "motivo": "Chuva prevista",
      "dia_anterior": "2026-10-18",
      "dia_novo": "2026-10-25",
      "created_at": "2026-09-09T18:30:00.000Z"
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | `Unauthorized` |
| 500 | Erro interno |

---

### GET /api/v1/campeonato-agendamentos/:id

Busca um agendamento pelo `id`. Aciona `ReadCampeonatoAgendamentoUsecase` (valida `id`; 404 senão).

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamento de campeonato encontrado com sucesso", "data": { "id": "665f...", "id_campeonato": "665a...", "data": "2026-10-18", "hora_inicio": "14:00", "hora_fim": "15:00", "court": "665c...", "unit": "665d...", "modalidade": "Beach Tênis", "status": "CONFIRMED" } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 | `Unauthorized` |
| 404 | `Agendamento de campeonato não encontrado` |

---

### PATCH /api/v1/campeonato-agendamentos/delete-bulk

Cancela agendamentos em massa. Aciona `DeleteBulkCampeonatoAgendamentosUsecase`. Segmento literal — registrado **antes** de `/:id`.

- **Sem `ids`** → cancela **todos** os agendamentos (`CONFIRMED`/`DRAFT`) do `id_campeonato`.
- **Com `ids`** → cancela só os informados.

Para cada agendamento que estava `CONFIRMED`, libera os `scheduling`s (`releaseEventFromSchedulings`). `DRAFT` não libera nada (nunca bloqueou).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id_campeonato` | string | Sim | ObjectId (24 hex) |
| `ids` | string[] | Não | ObjectIds (24 hex). Ausente = todos do campeonato |

**Resposta de sucesso** — `200 OK` (`data` = lista dos agendamentos afetados)
```json
{ "message": "Agendamentos de campeonato removidos com sucesso", "data": [ { "id": "665f...", "status": "CANCELLED", "...": "..." } ] }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` — `id_campeonato`/`ids` não são ObjectId |
| 401 | `Unauthorized` |
| 500 | Erro interno |

---

### PATCH /api/v1/campeonato-agendamentos/:id

Update **não-estrutural** — só `modalidade`. Aciona `UpdateCampeonatoAgendamentoUsecase`. Mudar `data` é via `trocar-dia`; `court`/`hora_*` não são editáveis (cancele e recrie).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `modalidade` | string | Não | Texto livre |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamento de campeonato atualizado com sucesso", "data": { "id": "665f...", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 | `Unauthorized` |
| 404 | `Agendamento de campeonato não encontrado` |

---

### PATCH /api/v1/campeonato-agendamentos/:id/delete

Cancela um agendamento. Aciona `DeleteCampeonatoAgendamentoUsecase`. `CONFIRMED` → `CANCELLED` + libera o `scheduling`; `DRAFT` → `CANCELLED` direto.

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Agendamento de campeonato removido com sucesso", "data": { "id": "665f...", "status": "CANCELLED", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `ID inválido` |
| 401 | `Unauthorized` |
| 404 | `Agendamento de campeonato não encontrado` |

---

### PATCH /api/v1/campeonato-agendamentos/:id/trocar-dia

Troca **somente o dia** (`data`) — quadra/horário/modalidade ficam intactos. Aciona `ChangeDayCampeonatoAgendamentoUsecase`:

1. Valida `motivo` (obrigatório) e `id`.
2. Lê o agendamento (404 senão); só `CONFIRMED` (400 senão).
3. `dia_novo` deve ser diferente do atual e não passado (400 senão).
4. Detecta conflitos no **novo** slot (bloqueadores + reservas ativas, excluindo o próprio agendamento).
   - Há conflito e `cancelar_conflitos` ≠ `true` → **409** `CampeonatoAgendamentoConflictError` (mesma forma do `POST`), **não move nada**.
   - `cancelar_conflitos: true` (ou sem conflito) → cancela os conflitantes, libera o slot antigo, atualiza `data`, aplica bloqueio no novo slot, grava auditoria.

> `usuario_nome` vem **no corpo** (rota interna, sem token Firebase) — é o MS de campeonatos quem sabe qual admin da UI dele disparou a ação.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dia_novo` | string | Sim | `"YYYY-MM-DD"`, diferente do atual, não passado |
| `motivo` | string | Sim | Texto livre — gravado na auditoria |
| `usuario_nome` | string | Sim | Nome do admin que disparou a troca (para a auditoria) |
| `cancelar_conflitos` | boolean | Não | `true` = cancela os conflitantes do novo dia e move |

**Resposta de sucesso** — `200 OK`
```json
{ "message": "Dia do agendamento de campeonato trocado com sucesso", "data": { "id": "665f...", "data": "2026-10-25", "hora_inicio": "14:00", "hora_fim": "15:00", "court": "665c...", "unit": "665d...", "modalidade": "Beach Tênis", "status": "CONFIRMED" } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` / `Motivo é obrigatório para trocar o dia` / `ID inválido` / `Só é possível trocar o dia de um agendamento CONFIRMED` / `O novo dia deve ser diferente do dia atual` / `Não é possível trocar para uma data passada` |
| 401 | `Unauthorized` |
| 404 | `Agendamento de campeonato não encontrado` |
| 409 | `CampeonatoAgendamentoConflictError` — novo dia tem conflito e `cancelar_conflitos` ≠ `true` |

**Exemplo de chamada**
```bash
curl -X PATCH http://agendamentos:5000/api/v1/campeonato-agendamentos/665f.../trocar-dia \
  -H "x-api-key: $AGENDAMENTOS_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "dia_novo": "2026-10-25", "motivo": "Chuva prevista", "usuario_nome": "Ana Admin", "cancelar_conflitos": true }'
```

## Referências

- Rota: `src/applications/routes/campeonato-agendamento.route.ts` (montada em `routes.ts` como `/campeonato-agendamentos`; `router.use(internalApiKeyMiddleware)`)
- Middleware: `src/applications/middlewares/internal-api-key.middleware.ts`
- Controllers: `src/applications/controllers/campeonato_agendamento/{create-bulk,read,list,list-auditoria,update,delete,change-day}/*.controller.ts`
- Usecases: `src/domain/usecases/campeonato-agendamento/{create-bulk,read,list,update,delete,change-day,list-auditoria}/*.usecase.ts` + `shared/campeonato-agendamento-conflict.error.ts`
- Serviços de domínio compartilhados: `src/domain/usecases/shared/{slot-allocator,conflict-resolution.service,event-conflict.service,event-scheduling-impact.service}.ts`
- Adapters: `src/infra/adapters/campeonato_agendamento/{create-many,read,list,update,set-status,change-data,delete-bulk,find-confirmed-by-court-unit}/*.adapter.ts` + `src/infra/adapters/campeonato_agendamento_auditoria/{create,list}/*.adapter.ts`
- Schemas: `src/infra/schemas/campeonato-agendamento.schema.ts` (coleção `campeonato_agendamentos`), `src/infra/schemas/campeonato-agendamento-auditoria.schema.ts` (coleção `campeonato_agendamento_auditorias`)
- DTOs: `src/applications/dto/campeonato-agendamento-bulk.dto.ts`, `src/applications/dto/campeonato-agendamento.dto.ts`
- Modelos: `src/domain/models/campeonato-agendamento.model.ts`, `src/domain/models/campeonato-agendamento-auditoria.model.ts`
- Ports: `src/domain/ports/input/campeonato-agendamento.input-port.ts`, `src/domain/ports/output/campeonato-agendamento-persistence.port.ts`, `src/domain/ports/output/campeonato-agendamento-shared.port.ts`, `src/domain/ports/output/campeonato-agendamento-auditoria-persistence.port.ts`
- Consumidor (outro serviço): `beach-center-bff-campeonatos` — ver [`campeonato.md`](../beach-center-bff-campeonatos/campeonato.md) e [`partida.md`](../beach-center-bff-campeonatos/partida.md)
- Recurso irmão: [`ranking-agendamento.md`](ranking-agendamento.md)
