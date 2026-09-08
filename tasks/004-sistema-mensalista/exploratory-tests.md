# Testes exploratórios manuais — 004 Sistema Mensalista + separação de Aula de `events_scheduled`

> Gerado por `/speckit-test`. Roteiro manual (o projeto não tem QA). Não altera código.
> Base: `context.md`, `plan.md` (AC-1 … AC-21). Complementa a suíte automatizada
> (`agendamentos` 1001 testes / `beach-center-bff-aulas` 187 testes) — os cenários abaixo cobrem
> **integração real entre serviços + MongoDB + fuso**, que os unit tests mockam.

---

## Escopo e pré-condições

### Serviços

| Serviço | Papel na task |
|---|---|
| `beach-center-bff-agendamentos` (`:5000`) | dono de `mensalistas`, `mensalista_planos`, `aula_bloqueios`, `schedulings`, `eventos_agendados` |
| `beach-center-bff-aulas` (`:5003`) | cria/atualiza/cancela `aula` → chama `agendamentos` via HTTP interno |
| `beach-center-server` (nginx `:8080`) | gateway; `docker-compose.dev.yml` sobe tudo |
| MongoDB | coleções `mensalistas`, `mensalista_planos`, `aula_bloqueios`, `schedulings`, `eventos_agendados`, `unidades`, `quadras`, `reservas` |

### Como subir

```
cd beach-center-server
cp .env.dev.example .env.dev            # opcional
./scripts/dev-up.sh                      # docker compose -f docker-compose.dev.yml up -d --build
```

- `aulas` precisa de `AGENDAMENTOS_API_URL=http://agendamentos:5000/api/v1` e
  `AGENDAMENTOS_INTERNAL_API_KEY=dev-agendamentos-key` (já no compose).
- Sem Docker: subir `agendamentos` e `aulas` com `npm run dev` em cada pasta, exportando as 2
  variáveis no ambiente do `aulas` apontando para `http://localhost:5000/api/v1`.

### Seed mínimo

| Item | Valor |
|---|---|
| 1 `unidade` U1 | com `courts: [Q1, Q2]` (2 quadras) |
| 1 `unidade` U2 | com `courts: [Q3]` |
| Usuário ADMIN | token Firebase válido (`Authorization: Bearer <token>`) |
| Usuário PROFESSOR P1 | `user_type: "PROFESSOR"` (para criar aula) |
| Usuário CLIENTE C1 | para tentativa de acesso negado |
| Fuso do host / container | `America/Sao_Paulo` (ou `TZ=America/Sao_Paulo`) |

### Perfis / auth

- `/mensalistas` e `/mensalistas/:id/planos` → `authMiddleware` + `requireRole("ADMIN")`.
- `/aula-bloqueios` → `internalApiKeyMiddleware` (`x-api-key: dev-agendamentos-key`), **sem** Firebase.
- `/eventos-agendados` → ADMIN.
- `/aulas` (task 002) → ADMIN cria; `requireOwnerOrAdmin` atualiza.

### Convenções

- "quadra livre no slot X" = nenhum `events_scheduled` OUTRO CONFIRMED, nenhum `mensalista_plano`
  CONFIRMED e nenhum `aula_bloqueio` CONFIRMED sobrepondo aquele dia-da-semana + janela.
- `available=false` deve ser conferido em `GET /api/v1/dias/:date/agendamentos` (list-day-schedulings)
  **e/ou** direto na coleção `schedulings`.

---

## 1. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **H-1** (AC-1) | ADMIN logado | `POST /api/v1/mensalistas` `{ "nome":"João", "telefone":"11999990000", "vencimento_fatura":"2099-12-31" }` | `201`; body `data.ativo === true`; documento em `mensalistas` com `deleted:false` |
| **H-2** (AC-1) | ADMIN | `POST /api/v1/mensalistas` com `vencimento_fatura` no **passado** (`"2020-01-01"`) | `201`; `data.ativo === false` |
| **H-3** (AC-4) | mensalista M1 de H-1 | `GET /api/v1/mensalistas/{M1}` ; `PATCH /api/v1/mensalistas/{M1}` `{ "telefone":"11888887777" }` | `GET` `200` com M1; `PATCH` `200`, telefone atualizado |
| **H-4** (AC-2) | M2 com `vencimento_fatura` passado mas `ativo:true` forçado no banco (`db.mensalistas.updateOne({_id:M2},{$set:{ativo:true}})`) | `GET /api/v1/mensalistas` | M2 volta com `ativo:false` **e** o banco foi atualizado (reconsultar o doc: `ativo:false`) |
| **H-5** (AC-3) | M2 com `ativo:false` | `PATCH /api/v1/mensalistas/{M2}` `{ "vencimento_fatura":"2099-01-01" }` → depois `GET /api/v1/mensalistas` | após o `GET`, M2 com `ativo:true` (recálculo lazy + o adapter de update já seta `ativo` na hora) |
| **H-6** (AC-5) | M1 existe; Q1 livre em seg/qua 08:00–09:00; existem `schedulings` de segundas/quartas futuras 08:00–09:00 em Q1 | `POST /api/v1/mensalistas/{M1}/planos` `{ "dias":["segunda","quarta"], "start_time":"08:00", "end_time":"09:00", "court":"Q1", "modalidade":"Beach Tenis", "equipamentos_proprios":false }` | `201`; `data.status==="CONFIRMED"`; `data.unit` = U1 (derivado de Q1); **todos** os `schedulings` seg/qua 08:00–09:00 de Q1 de hoje em diante ficam `available:false` |
| **H-7** (AC-7) | plano de H-6 CONFIRMED | criar um novo `scheduling` de uma **segunda futura** 08:00–09:00 em Q1 (via `POST /api/v1/agendamentos` ou abrindo o dia em `GET /dias/:date/agendamentos`) | o novo `scheduling` nasce `available:false` |
| **H-8** (AC-9) | plano de H-6 CONFIRMED, sem reserva ativa nos slots; nenhum outro bloqueador | `PATCH /api/v1/mensalistas/{M1}/planos/{P1}/delete` | `200`; `data.status==="CANCELLED"`; os `schedulings` seg/qua 08:00–09:00 de Q1 (futuros, sem reserva) voltam a `available:true` |
| **H-9** (AC-11) | M3 com 2 planos CONFIRMED (P-a seg 07:00–08:00 Q1, P-b sex 19:00–20:00 Q2) | `PATCH /api/v1/mensalistas/{M3}/delete` | `200`; M3 `deleted:true`; P-a e P-b `status:"CANCELLED"`; slots de seg 07:00 Q1 e sex 19:00 Q2 (sem reserva, sem outro bloqueador) voltam a `available:true` |
| **H-10** (AC-12) | ADMIN em `aulas`; P1 é PROFESSOR; Q3 livre em ter 19:00–20:00 | `POST /api/v1/aulas` `{ "classe":"T-A", "modalidade":"Beach Tenis", "dias":["terça"], "hora_inicio":"2026-01-06T22:00:00.000Z", "hora_fim":"2026-01-06T23:00:00.000Z", "professor":"P1", "quadra":"Q3", "capacidade_maxima":10 }` (22:00Z = 19:00 America/Sao_Paulo) | `201`; **e** em `agendamentos`: `db.aula_bloqueios` tem 1 doc `{ aula_id:<id>, dias:["terça"], start_time:"19:00", end_time:"20:00", court:Q3, unit:U2, status:"CONFIRMED" }`; `schedulings` de terças futuras 19:00–20:00 de Q3 ficam `available:false` |
| **H-11** (AC-14) | aula A1 de H-10 com bloqueio CONFIRMED | `PUT /api/v1/aulas/{A1}` mudando `quadra` para Q2 (livre no slot) | `200`; o `aula_bloqueio` antigo é cancelado/recriado apontando para Q2 (checar `db.aula_bloqueios`: 1 CONFIRMED em Q2, o anterior em Q3 `CANCELLED`); slots de Q3 ter 19:00 liberados, slots de Q2 ter 19:00 bloqueados |
| **H-12** (AC-14) | aula A1 com bloqueio CONFIRMED | `DELETE /api/v1/aulas/{A1}` | `200`; `aula_bloqueio` da aula fica `CANCELLED`; slots liberados (sem reserva/outro bloqueador) |
| **H-13** (AC-16) | ADMIN em `agendamentos` | `POST /api/v1/eventos-agendados` `{ "event_type":"OUTRO", "day_of_week":"sábado", "start_time":"10:00", "end_time":"12:00", "court":"Q1", "unit":"U1" }` | `201` (OUTRO continua funcionando); slots de sábado 10:00–12:00 de Q1 → `available:false` |
| **H-14** (AC-17) | 1 dia com Q1 bloqueada por `mensalista_plano` (H-6), Q2 por `aula_bloqueio`, Q1 por `events_scheduled OUTRO` (H-13) no mesmo dia | `GET /api/v1/dias/{date}/agendamentos` | `exception_conflicts[].exceptions[]` traz os três; cada um com `source` (`"MENSALISTA"`, `"AULA"`, `"EVENT"`); shape anterior preservado + campo `source` |
| **H-15** (AC-15) | — | `GET /api/v1/aula-bloqueios` **com** `x-api-key: dev-agendamentos-key` | `200` lista |
| **H-16** (AC-19) | N docs `eventos_agendados` `event_type:"MENSALISTA"` (inserir 3 à mão, 1 CANCELLED) | `npm run migrate:mensalista` (com `DB` apontando pro banco) | log "3 criado(s)"; `mensalistas` +3 placeholder (`telefone` placeholder, `vencimento_fatura` 2999, `ativo:true`); `mensalista_planos` +3 (`source_event_id` = `_id` do evento; `status` herdado; `dias:[day_of_week]`); os 3 `eventos_agendados` continuam lá |
| **H-17** (AC-20) | N docs `eventos_agendados` `AULA_BEACH_TENIS`/`AULA_VOLEI` (inserir 2) | `npm run migrate:aula-bloqueio` | log "2 criado(s)"; `aula_bloqueios` +2 (**sem** `aula_id`; `modalidade` `"Beach Tênis"`/`"Vôlei"`; `source_event_id`); originais intactos |

---

## 2. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **E-1** (AC-6a) | Q1 já tem `events_scheduled OUTRO` CONFIRMED seg 08:00–09:00 | `POST /mensalistas/{M1}/planos` com `dias:["segunda"]` 08:00–09:00 Q1 | `400` (conflito, `MensalistaPlanoConflictError`); nada criado em `mensalista_planos` |
| **E-2** (AC-6b) | Q1 já tem `mensalista_plano` CONFIRMED seg 08:00–09:00 (outro mensalista) | `POST /mensalistas/{M1}/planos` mesmo slot | `400`; nada criado |
| **E-3** (AC-6c / AC-18) | Q1 já tem `aula_bloqueio` CONFIRMED ter 19:00–20:00 | `POST /mensalistas/{M1}/planos` `dias:["terça"]` 19:00–20:00 Q1 | `400`; nada criado |
| **E-4** (AC-18 inverso) | Q2 tem `mensalista_plano` CONFIRMED qua 20:00–21:00 | `POST /aula-bloqueios` (`x-api-key`) `{ "dias":["quarta"], "start_time":"20:00", "end_time":"21:00", "court":"Q2", "modalidade":"Volei" }` | `409` (`AulaBloqueioConflictError`); nada criado em `aula_bloqueios` |
| **E-5** (AC-18 inverso) | Q2 tem `aula_bloqueio` CONFIRMED qui 18:00–19:00 | `POST /eventos-agendados` `event_type:"OUTRO"` qui 18:00–19:00 Q2 | `400` (`EventConflictError`) |
| **E-6** (AC-13) | Q3 já tem `aula_bloqueio` CONFIRMED ter 19:00–20:00 | `POST /api/v1/aulas` para P1 em Q3 terça 19:00–20:00 | `agendamentos` responde `409` ao client → `create-aula` faz **rollback**: `db.aulas` **não** tem a aula (hard-delete); `db.aula_bloqueios` **não** ganhou registro pendurado; resposta HTTP `409` (mapeada de `ConflictError`) |
| **E-7** (AC-16) | ADMIN | `POST /eventos-agendados` `event_type:"MENSALISTA"` (e repetir com `"AULA_BEACH_TENIS"`, `"AULA_VOLEI"`) | `400` com mensagem apontando para `/mensalistas` / serviço de aulas; nada criado |
| **E-8** (AC-16) | ADMIN; existe 1 `events_scheduled` `AULA_VOLEI` (legado ou migrado) | `PATCH /eventos-agendados/{id}` com `event_type:"AULA_VOLEI"` | `400` (update também rejeita ≠ OUTRO) |
| **E-9** (AC-16) | há `events_scheduled` `MENSALISTA` legados | `GET /api/v1/eventos-agendados?event_type=MENSALISTA` | `200` — consulta de legados **continua permitida** (só create/update bloqueiam) |
| **E-10** (AC-15) | — | `PATCH /api/v1/aula-bloqueios/{qualquer}/delete` **sem** header `x-api-key` | `401` |
| **E-11** (AC-15) | ADMIN com token Firebase válido, **sem** `x-api-key` | qualquer método de `/api/v1/aula-bloqueios` | `401` — Firebase **não** substitui a `x-api-key` (rota é `internalApiKeyMiddleware` puro) |
| **E-12** (AC-15) | `x-api-key` **errada** | `GET /api/v1/aula-bloqueios` `x-api-key: xpto` | `401` |
| **E-13** (auth) | usuário CLIENTE C1 (não ADMIN) | `POST /api/v1/mensalistas` | `403` |
| **E-14** (validação) | ADMIN | `POST /mensalistas/{M1}/planos` `{ "dias":[], "start_time":"08:00", "end_time":"09:00", "court":"Q1", "modalidade":"x", "equipamentos_proprios":false }` | `400` (`dias` min 1) |
| **E-15** (validação) | ADMIN | `POST /mensalistas/{M1}/planos` com `start_time:"10:00"`, `end_time:"09:00"` | `400` (`start_time` deve ser anterior ao `end_time`) |
| **E-16** (validação) | ADMIN | `POST /mensalistas/{M1}/planos` com `start_time:"8h"` ou `court:"abc"` | `400` (regex `HH:MM` / ObjectId) |
| **E-17** (not found) | ADMIN | `POST /mensalistas/{id-inexistente}/planos` (payload válido) | `404` ("Mensalista não encontrado") |
| **E-18** (not found) | ADMIN | `POST /mensalistas/{M1}/planos` com `court` = ObjectId válido de quadra **inexistente / de nenhuma unidade** | `404` ("Quadra não encontrada") |
| **E-19** (integração fora do ar) | subir `aulas` **sem** `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY` | `POST /api/v1/aulas` (payload válido) | `500` com mensagem "Integração com agendamentos não configurada…"; aula sofre rollback hard-delete (não fica órfã) |
| **E-20** (integração fora do ar) | `aulas` de pé, `agendamentos` **derrubado** | `POST /api/v1/aulas` | erro de rede no client → rollback hard-delete + erro propagado (`500`); nenhuma aula persistida |
| **E-21** (integração fora do ar) | `aulas` de pé, `agendamentos` derrubado | `DELETE /api/v1/aulas/{A}` (aula existente) | soft-delete acontece, depois o cancel do bloqueio falha → resposta `500` (**bloqueante**, por design); reexecutar o `DELETE` depois que `agendamentos` voltar deve cancelar o bloqueio |
| **E-22** (cancel por aula) | aula A com bloqueio CONFIRMED | `PATCH /api/v1/aula-bloqueios/by-aula/{A}/delete` (`x-api-key`) | `200`; body `data` = lista dos bloqueios cancelados; `db.aula_bloqueios` daquela aula → `CANCELLED` |
| **E-23** (cancel sem alvo) | — | `PATCH /api/v1/aula-bloqueios/none/delete` sem `?aula_id` e `:id` não-ObjectId | `400`/`404` conforme validação (`assertValidObjectId` no `id`) |

---

## 3. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **G-1** (fuso) | host/container `America/Sao_Paulo` | criar aula com `hora_inicio` ISO **com `Z`** representando 22:00Z | `aula_bloqueio.start_time` = `"19:00"` (conversão UTC→America/Sao_Paulo pelo `horaToHHMM`). ⚠️ Se o front enviar "20:00 parede" como `20:00Z`, o bloqueio vira `17:00` — **documentar com o time** qual é o contrato de `hora_inicio` |
| **G-2** (arredondamento) | — | `POST /mensalistas/{M1}/planos` com `start_time:"08:15"`, `end_time:"08:50"` | o usecase aplica `formatRoundedTime('start')`/`('end')` → conferir `data.start_time`/`data.end_time` arredondados como em `events_scheduled` (start p/ baixo, end p/ cima) |
| **G-3** (inadimplência não libera — AC-8) | `mensalista_plano` CONFIRMED cujo mensalista está `ativo:false` (vencimento vencido) | `GET /api/v1/dias/{date}/agendamentos` para os dias/horário do plano | slots **continuam** `available:false` — só o `PATCH .../planos/:id/delete` libera; `ativo:false` é informativo |
| **G-4** (cancel não libera slot com reserva — AC-10) | slot bloqueado por P1; criar uma **reserva ativa** nesse slot | `PATCH .../planos/{P1}/delete` | plano `CANCELLED`, mas o slot com reserva ativa **permanece** `available:false` |
| **G-5** (cancel não libera slot bloqueado por outro — AC-10) | slot coberto por **2** planos sobrepostos (P1 e P2) | `PATCH .../planos/{P1}/delete` | slot **permanece** `available:false` (P2 ainda CONFIRMED; o `release` re-checa `hasConflict` que enxerga as 3 fontes) |
| **G-6** (normalização de dia) | — | criar plano com `dias:["Segunda-feira"]` e outro com `dias:["segunda"]` no mesmo slot | ambos normalizam para o mesmo dia-da-semana → o 2º dá `400` (conflito) |
| **G-7** (update estrutural de plano proibido) | plano CONFIRMED | `PATCH /mensalistas/{M1}/planos/{P1}` `{ "dias":["sexta"], "modalidade":"Volei" }` | só `modalidade`/`equipamentos_proprios`/`price` mudam; `dias` **ignorado** (não editável). Para trocar dias/quadra/horário: cancelar e recriar |
| **G-8** (update estrutural de aula_bloqueio) | `aula_bloqueio` B CONFIRMED seg 08:00–09:00 Q1 | `PATCH /api/v1/aula-bloqueios/{B}` (`x-api-key`) `{ "dias":["segunda","quarta"], "start_time":"08:00", "end_time":"09:00", "court":"Q1", "modalidade":"Beach Tenis" }` | `200`; conflito revalidado **excluindo o próprio B** (`excludeBloqueioId`); slots de segunda permanecem bloqueados, quartas passam a bloqueadas |
| **G-9** (price) | — | criar plano com `price:0` e outro com `price:-1` | `price:0` aceito; `price:-1` → `400` (`min(0)`) |
| **G-10** (migração idempotente) | rodar `migrate:mensalista` **duas vezes** seguidas | 2ª execução | log "0 criado(s), N já migrado(s)"; sem duplicatas em `mensalistas`/`mensalista_planos` |
| **G-11** (migração idempotente) | rodar `migrate:aula-bloqueio` **duas vezes** | 2ª execução | log "0 criado(s), N já migrado(s)"; sem duplicatas |
| **G-12** (migração — evento CANCELLED) | `eventos_agendados` `MENSALISTA` com `status:"CANCELLED"` | `migrate:mensalista` | cria `mensalista_plano` com `status:"CANCELLED"` → **não** bloqueia quadra nenhuma (apply só roda para CONFIRMED) |
| **G-13** (aula com vários dias) | criar aula `dias:["segunda","quarta","sexta"]` | após `POST /aulas` | `aula_bloqueio` único com `dias` = os 3; `EventConflictService`/impact "explodem" em 3 no `find-confirmed-by-court-unit` |
| **G-14** (reorder de dias no update de aula) | aula A `dias:["segunda","quarta"]` | `PUT /aulas/{A}` `dias:["quarta","segunda"]` (só reordenou) | `update-aula` **não** re-sincroniza o bloqueio (comparação ordena antes) — nenhum cancel/create no client |
| **G-15** (slot passado) | plano CONFIRMED; existe `scheduling` de um dia **já passado** no mesmo dia-da-semana | cancelar o plano | slots passados **não** são reabertos (`isPastScheduling`) |

---

## 4. Checklist de regressão (fluxos vizinhos)

| ID | O que checar | Resultado esperado |
|---|---|---|
| **R-1** | `events_scheduled` `OUTRO`: create / update / delete / list | inalterado — CRUD de OUTRO funciona 100% (só MENSALISTA/AULA_* barrados no create/update) |
| **R-2** | `EventConflictService` sem as 2 fontes novas injetadas (teoria) / com elas vazias | matching idêntico ao anterior — criar `scheduling`/`events_scheduled` num slot livre continua nascendo `available:true` |
| **R-3** | `EventSchedulingImpactService.releaseEventFromSchedulings` | re-checa `hasConflict` contra as **3** fontes; um slot coberto por evento OUTRO + plano cancelado permanece bloqueado |
| **R-4** | `list-day-schedulings` — painel "Exceções" do front | `exception_conflicts` mantém o shape; front que ignora `source` continua funcionando; front atualizado lê `source` |
| **R-5** | `create/update-scheduling` (novo slot) | continua consultando `hasConflict` (agora 3 fontes) — slot que cai sobre um `mensalista_plano`/`aula_bloqueio` nasce `available:false` |
| **R-6** | `reserve-scheduling` validator | reservar um slot bloqueado por plano/bloqueio → rejeitado (mesma regra de conflito) |
| **R-7** | Aula (task 002) — `POST /aulas` em quadra **livre** | continua `201`; agora **também** cria o `aula_bloqueio` (efeito colateral novo) |
| **R-8** | Aula — `PUT /aulas/{id}` mudando só `classe`/`capacidade_maxima`/`professor` | `200`; **nenhuma** chamada ao client de bloqueio (não é campo estrutural) |
| **R-9** | Aluno (task 002) — CRUD de `/aulas/:id/alunos` | inalterado |
| **R-10** | `agendamentos → pagamentos` (refund) | inalterado — o novo client `aulas → agendamentos` usa o mesmo padrão mas não toca esse fluxo |
| **R-11** | `docker-compose.dev.yml` | `docker compose -f docker-compose.dev.yml config` válido; serviço `aulas` sobe com as 2 novas envs; healthchecks verdes |
| **R-12** | `beach-center-app` (front) | sem mudanças nesta task; o painel "Exceções" pode exibir `source` cru até ajuste futuro (fora de escopo) |

---

## 5. Rastreabilidade AC → cenário

| AC | Cenários | Cobertura automatizada equivalente? |
|---|---|---|
| AC-1 (criar mensalista + ativo derivado) | H-1, H-2 | ✅ `create-mensalista.usecase.spec` + controller |
| AC-2 (recálculo lazy) | H-4, R-2 | ✅ `list-mensalistas.usecase.spec`, `recalculate-mensalistas-status.adapter.spec` |
| AC-3 (reativação) | H-5 | ✅ `update-mensalista.adapter.spec` (seta `ativo`) |
| AC-4 (read/update/delete) | H-3, H-9 | ✅ usecases + controllers |
| AC-5 (plano bloqueia quadra) | H-6 | ⚠️ parcial — usecase mockado; **efeito no `schedulings` real só aqui** |
| AC-6 (plano conflitante) | E-1, E-2, E-3 | ✅ `create-mensalista-plano.usecase.spec` (mock `hasConflict`) |
| AC-7 (slot futuro nasce bloqueado) | H-7 | ⚠️ **só manual** — geração lazy + impact com Mongo real |
| AC-8 (inadimplência não libera) | G-3 | ⚠️ **só manual** |
| AC-9 (cancelar libera) | H-8 | ✅ `cancel-mensalista-plano.usecase.spec` (release por dia) |
| AC-10 (não libera c/ reserva ou outro bloqueador) | G-4, G-5 | ⚠️ parcial — regra do `release` é real; unit cobre o "não estava CONFIRMED" |
| AC-11 (deletar mensalista cancela planos) | H-9 | ✅ `delete-mensalista.usecase.spec` + `cancel-...-by-mensalista.adapter.spec` |
| AC-12 (criar aula → bloqueio) | H-10, G-13 | ⚠️ **só manual** — HTTP interno real; unit cobre a chamada ao client |
| AC-13 (quadra ocupada → rollback) | E-6 | ✅ `create-aula.usecase.spec` (rollback + propaga); `create-aula-bloqueio-client.adapter.spec` (409→ConflictError) |
| AC-14 (update/delete aula reflete no bloqueio) | H-11, H-12, G-8, G-14, R-8 | ✅ `update-aula.usecase.spec`, `delete-aula.usecase.spec`, `update/cancel-aula-bloqueio.usecase.spec` |
| AC-15 (rota interna) | H-15, E-10, E-11, E-12 | ⚠️ `internalApiKeyMiddleware` já tem spec; combinação Firebase-sem-x-api-key **só manual** |
| AC-16 (events_scheduled só OUTRO) | H-13, E-7, E-8, E-9, R-1 | ✅ `create/update-events-scheduled.usecase.spec` |
| AC-17 (3 fontes no list-day) | H-14, R-4 | ✅ `event-conflict.service.spec` (merge 3 fontes), `list-day-schedulings.usecase.spec` (`source`) |
| AC-18 (conflito unificado 3 direções) | E-3, E-4, E-5 | ✅ `event-conflict.service.spec` + specs dos 3 usecases de create |
| AC-19 (migração mensalista idempotente) | H-16, G-10, G-12 | ✅ `migrate-mensalista-from-events.spec` |
| AC-20 (migração aula idempotente) | H-17, G-11 | ✅ `migrate-aula-bloqueio-from-events.spec` |
| AC-21 (isolamento + specs verdes + cobertura) | R-2, R-3, R-11 | ✅ ESLint + `tsc` + Jest (1001 / 187) + cobertura 93.5% / 91% branch |

**Cenários que dependem exclusivamente de execução manual** (Mongo + HTTP interno + fuso reais, sem
equivalente automatizado): **H-6, H-7, H-8, H-9, H-10, H-11, H-12, H-14, H-16, H-17, E-6, E-19,
E-20, E-21, E-22, G-1, G-2, G-3, G-4, G-5, G-15**.

---

## 6. Contagem

| Categoria | Qtd |
|---|---|
| Caminhos felizes (H) | 17 |
| Fluxos de exceção (E) | 23 |
| Edge cases (G) | 15 |
| Regressão (R) | 12 |
| **Total** | **67** |

Todos os 21 Critérios de Aceite rastreados a ≥ 1 cenário.
