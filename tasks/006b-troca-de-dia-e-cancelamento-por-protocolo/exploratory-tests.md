# Testes Exploratórios Manuais — Task 006b (Troca de dia e cancelamento público por protocolo)

> Gerado por `/speckit-test`. Roteiro de QA manual (o projeto não tem analistas de QA dedicados).
> **Não** contém código de teste automatizado — esses estão em `/speckit-unit-tests`
> (`agendamentos` 292 suítes / 1396 testes; `campeonatos` 83 / 331; `beach-center-whatsapp` **sem
> suíte**, decisão do usuário).
> Base: `plan.md` (Critérios de Aceite **AC-1 … AC-33**) + `context.md` (A1/B-btn/D3/D3b/D8/D9/
> D-resched/D-audit/H/Q1–Q5).
> **Parte 2 de 2.** Depende da **task 006a** (máquina de estados `pending → waiting_approve →
> approved → expired`, protocolo tardio, fim do reembolso).

---

## 1. Escopo e pré-condições

### 1.1 Serviços envolvidos

| Serviço | Papel no teste | Auth | Base URL |
|---|---|---|---|
| `beach-center-bff-agendamentos` | **Dono do fluxo.** Rotas públicas por protocolo (reserva + `ranking_agendamento`) de troca de dia e cancelamento; janela de 2 h no público **e** nas rotas internas de `trocar-dia` (task 005); auditoria de reagendamento; disparo de WhatsApp. | Rotas por protocolo: **públicas** (sem token). `trocar-dia` interno: `x-api-key`. | `http://localhost:<PORT_AG>/api/v1` |
| `beach-center-whatsapp` | `POST /messages/send-by-key` `{ to, key, vars }` — resolve o template e envia (Meta Cloud API). Seed dos `whatsapp_message_templates` no boot. Serviço **JS**, sem hexagonal/testes (decisão do usuário). | `x-api-key` = `WHATSAPP_INTERNAL_API_KEY` | `http://localhost:5004/api/v1` |
| `beach-center-bff-campeonatos` | `POST /rankings/:id/agendar` passa `contato: { nome, email, telefone }` no `create-bulk`; propaga o `409` da janela de 2 h. | Firebase + `role=ADMIN` | `http://localhost:<PORT_CAMP>/api/v1` |
| `beach-center-bff-usuarios` | Emissão de token ADMIN (só para `campeonatos` e para as rotas admin de `agendamentos`, não para as públicas). | — | — |
| ~~`beach-center-app`~~ | **NÃO tocado.** Página `/protocolo/:number`, modal com 2 opções e aviso de reembolso = task de frontend futura. | — | — |
| ~~`beach-center-bff-pagamentos`~~ | **NÃO tocado** nesta task. | — | — |

> ⚠️ **Correção da fase de validação:** o **`motivo` do cancelamento público é OPCIONAL**
> (reserva e `ranking_agendamento`) — o front atual cancela sem corpo. O **`motivo` do
> reagendamento é OBRIGATÓRIO** (`≤ 280` chars) e vai para a auditoria.

### 1.2 Ambiente

- Subir com `docker-compose.dev.yml` (task 003). Serviço `whatsapp` em `PORT 5004`,
  `WHATSAPP_INTERNAL_API_KEY: dev-whatsapp-key`. `agendamentos` recebe
  `WHATSAPP_API_URL: http://whatsapp:5004/api/v1` + `WHATSAPP_INTERNAL_API_KEY`.
- Mongo limpo (ou staging isolado).
- **Envio real de WhatsApp**: só funciona com `WHATSAPP_ACCESS_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID`
  válidos (Meta). Sem eles, `send-by-key` responde `500` mas o cancelamento/reagendamento
  **conclui** (não-bloqueante). Testar **os dois modos**:
  1. sem credenciais Meta → verificar que a operação principal conclui e o erro é só logado;
  2. com credenciais + número de teste → verificar a mensagem recebida.
- **Manipulação de tempo (janela de 2 h)**: criar o `scheduling` / `ranking_agendamento` com
  `hora_inicio` calculado a partir de "agora + X h" no fuso `-03:00`, ou ajustar o relógio do
  container.

### 1.3 Dados de seed

| Item | Detalhe |
|---|---|
| Unidade + quadras | 1 `unit` + 2 `court` ativos. |
| `scheduling` (dia futuro) | Slots gerados para **hoje+2 dias** e **hoje+3 dias** (via `GET /dias/:data/agendamentos` ou `create-day`), disponíveis. Pelo menos: 1 slot começando em **> 2 h** e 1 começando em **< 2 h** para a mesma quadra hoje. |
| Reserva `waiting_approve` | Criar reserva `pending` (`POST /reservas`), enviar comprovante em `pagamentos` (`POST /public/transaction-history`) → vira `waiting_approve` **com `number` de 10 dígitos** (006a). |
| Reserva `approved` | A anterior, aprovada pelo admin (`PATCH /transaction-history/:id/review`). |
| `ranking_agendamento` | Via `POST /rankings/:id/agendar` (campeonatos) com `contato: { nome, email, telefone }` → nasce `pending` com `numero_protocolo` e `cliente`. Anexar comprovante (`PATCH /ranking-agendamentos/protocolo/:n/comprovante`) → `waiting_approve`. Um segundo RA **sem** `contato` (legado). |
| Telefone de teste | Número real habilitado no WhatsApp Business de teste (para os cenários com Meta configurado). |

### 1.4 Perfis / permissões

| Ação | Quem |
|---|---|
| `GET /reservas/protocol/:number`, `PATCH .../cancel`, `PATCH .../reagendar` | **público** |
| `GET /ranking-agendamentos/protocolo/:n`, `PATCH .../cancelar`, `PATCH .../reagendar` | **público** |
| `PATCH /ranking-agendamentos/:id/trocar-dia`, `PATCH /campeonato-agendamentos/:id/trocar-dia` | `x-api-key` interna (chamado por `campeonatos`) |
| `POST /rankings/:id/agendar` | ADMIN (Firebase) via `campeonatos` |
| `POST /api/v1/messages/send-by-key` | `x-api-key` = `WHATSAPP_INTERNAL_API_KEY` |

---

## 2. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **H-1** (AC-2) | Reserva R `waiting_approve`; slot mais cedo começa em > 2 h | `GET /reservas/protocol/:R.number` | `200`. `data.can_cancel = true`, `data.can_reschedule = true`, `cancellation_deadline == reschedule_deadline` (2 h antes do slot mais cedo, ISO). Dados da reserva presentes. |
| **H-2** (AC-4) | RA `waiting_approve` (com `cliente`); jogo em > 2 h | `GET /ranking-agendamentos/protocolo/:numero_protocolo` | `200` com dados do RA + `cliente` + `can_cancel`/`can_reschedule = true` + deadlines. |
| **H-3** (AC-5) | Reserva R `waiting_approve`, slot em > 2 h | `PATCH /reservas/protocol/:R.number/cancel` **sem corpo** (ou `{ "motivo": "não vou mais" }`) | `200`. `R.status = "cancelled"`; `refund_status` **não** setado (não era `approved`); slots liberados; **nenhuma** chamada ao `whatsapp`. |
| **H-4** (AC-6, AC-7) | Reserva R `approved`, `total > 0`, slot em > 2 h; Meta **não** configurada | `PATCH /reservas/protocol/:R.number/cancel` | `200`. `R.status = "cancelled"`, `R.refund_status = "manual"`, slots liberados. **`agendamentos` chama `POST {WHATSAPP_API_URL}/messages/send-by-key`** com `key: "cancelamento-reembolso"`, `to = R.phone`, `vars.protocolo = R.number`. O `whatsapp` responde `500` (sem token Meta) → o cancelamento **conclui igual** e o erro só aparece no log de `agendamentos`. |
| **H-5** (AC-6 com Meta) | Idem H-4, mas Meta configurada + número de teste | Cancelar por protocolo | O celular de teste recebe: *"Ola, {nome}! Sua reserva {protocolo} foi cancelada… responda esta conversa e nossa equipe cuida do reembolso…"* |
| **H-6** (AC-10) | RA `approved` **com `cliente.telefone`**, jogo em > 2 h | `PATCH /ranking-agendamentos/protocolo/:n/cancelar` (sem corpo) | `200`. `status = "cancelled"`; o bloqueio da janela exata é liberado; `send-by-key` chamado com `key: "cancelamento-reembolso"`, `to = cliente.telefone`. |
| **H-7** (AC-13) | Reserva R `waiting_approve` de **1 slot**, slot atual em > 2 h; existe um `scheduling` disponível em hoje+3 (`{unit, court, date, start_time, end_time}` batendo), começando em > 2 h, sem conflito | `PATCH /reservas/protocol/:R.number/reagendar` com `{ "motivo": "vou viajar", "slots": [<slot novo>] }` | `200`. `R.scheduling_id` aponta para o novo `scheduling`; slot antigo volta a `available: true`; novo fica `available: false`; **`R.number` e `R.total` inalterados**. 1 registro em `reserva_reagendamento_auditorias` com `slots_anteriores`/`slots_novos`/`motivo`. `send-by-key` `key: "reagendamento-confirmado"` para `R.phone`. |
| **H-8** (AC-14) | Reserva R `approved` de **2 slots** | Reagendar com **2** slots novos válidos | `200`. Os 2 antigos liberados, os 2 novos vinculados; `R.total` inalterado; auditoria com 2+2 slots. |
| **H-9** (AC-22) | RA `waiting_approve` com `cliente`, jogo em > 2 h; `scheduling` novo disponível/sem conflito | `PATCH /ranking-agendamentos/protocolo/:n/reagendar` com `{ motivo, unit, court, date, start_time, end_time }` | `200`. O RA passa a ter `court`/`unit`/`data`/`hora_inicio`/`hora_fim` novos; bloqueio da janela antiga liberado, o da nova aplicado; `numero_protocolo` inalterado; registro em `ranking_agendamento_auditorias` com `slot_anterior`/`slot_novo` e `usuario_nome = cliente.nome`; `send-by-key` `reagendamento-confirmado` para `cliente.telefone`. |
| **H-10** (AC-27) | `campeonato_agendamento` / RA `CONFIRMED`/`pending` cujo jogo é em **hoje+5 dias**, `motivo` informado | `PATCH /campeonato-agendamentos/:id/trocar-dia` (ou `/ranking-agendamentos/:id/trocar-dia`), `x-api-key`, com `dia_novo` válido | `200` — comportamento da task 005 preservado (troca a `data`, grava auditoria, fluxo de conflito). |
| **H-11** (AC-28) | Ranking `PARTIDAS` com N partidas pendentes | `POST /rankings/:id/agendar` com `contato: { nome, email, telefone }` | `201`. Cada `ranking_agendamento` criado (via `GET /ranking-agendamentos?id_ranking=`) tem `cliente` == `contato`. |
| **H-12** (AC-32) | Template `reagendamento-confirmado` no banco (seed) | `POST /api/v1/messages/send-by-key` `{ to, key: "reagendamento-confirmado", vars: { nome, protocolo, novo_dia, novo_horario, quadra } }` com `x-api-key` correta | `200`. Com Meta configurada, a mensagem chega com as `vars` interpoladas. |

---

## 3. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **E-1** (AC-1) | Reserva `pending` (sem `number`, ou com `number` legado) | `GET /reservas/protocol/:number` | `404`. (`pending` não tem protocolo ativo — B-btn.) |
| **E-2** (AC-4) | RA `pending` (comprovante ainda não enviado) | `GET /ranking-agendamentos/protocolo/:numero_protocolo` | `404` (`"Agendamento de ranking não encontrado"`). |
| **E-3** (AC-3) | Reserva `cancelled` / `rejected` / `expired` (com `number`) | `GET /reservas/protocol/:number` | `200` com `can_cancel = false`, `can_reschedule = false`; `*_deadline` ainda devolvidos se houver slots. |
| **E-4** (AC-8) | Reserva `waiting_approve` cujo slot mais cedo começa em **< 2 h** | `PATCH /reservas/protocol/:number/cancel` | `409`/`400` — `"Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"`. Nada muda. Sem WhatsApp. |
| **E-5** (AC-9) | Reserva `cancelled` | `PATCH /reservas/protocol/:number/cancel` | `409` `"Reserva ja foi cancelada"`. Reserva `expired` → `409` `"Reserva expirada"`. |
| **E-6** (AC-1) | Reserva `pending` | `PATCH /reservas/protocol/:number/cancel` | `404` (B-btn — ação pública só a partir de `waiting_approve`). |
| **E-7** (AC-24) | RA `cancelled`/`rejected`/`expired` | `PATCH /ranking-agendamentos/protocolo/:n/cancelar` | `409`. RA `pending` → `404`. |
| **E-8** (AC-15) | Reserva de **2** slots | Reagendar enviando **1** (ou **3**) slots | `409` `"O reagendamento deve manter o mesmo numero de horarios"`. Nada muda. |
| **E-9** (AC-16) | Slot novo cujo `scheduling` **não existe** (dia não provisionado), **ou** existe mas `available: false` (dia fechado / já reservado) | `PATCH /reservas/protocol/:number/reagendar` | `409` `"Horario indisponivel para reagendamento"`. Slots antigos **permanecem** vinculados e bloqueados. |
| **E-10** (AC-17) | Slot novo que coincide com `aula_bloqueio` / `mensalista_plano` / `campeonato_agendamento` / `ranking_agendamento` ativo, ou evento `OUTRO` `CONFIRMED` | Reagendar | `409` `"Um ou mais horarios selecionados possuem excecao de agendamento"`. Nada muda. |
| **E-11** (AC-18) | Slot novo com data já passada, **ou** começando em **< 2 h** de agora | Reagendar | `400` `"...datas que ja passaram"` / `409` `"Novo horario deve comecar ao menos 2 horas a partir de agora"`. Nada muda. |
| **E-12** (AC-19) | Reserva cujo slot atual mais cedo começa em **< 2 h** | `PATCH /reservas/protocol/:number/reagendar` | `409` (fora da janela dos slots atuais). Nada muda. Sem WhatsApp. |
| **E-13** (AC-20) | Reserva `cancelled`/`rejected`/`expired` → reagendar → `409`. Reserva `pending` → reagendar → `404`. | | |
| **E-14** (motivo do reagendamento) | Reserva `waiting_approve` válida | `PATCH .../reagendar` com `{ "slots": [...] }` **sem `motivo`** (ou `motivo` só com espaços, ou > 280 chars) | `400` `"Dados invalidos"` / `"Motivo e obrigatorio"`. Nada muda. |
| **E-15** (AC-23) | RA, slot novo com bloqueador ou reserva ativa (`available: false`) | `PATCH /ranking-agendamentos/protocolo/:n/reagendar` | `409` (`RankingAgendamentoConflictError`). Nada muda. |
| **E-16** (AC-25) | RA cujo `data` + `hora_inicio` está a **< 2 h** de agora | `PATCH /ranking-agendamentos/:id/trocar-dia` (`x-api-key`) | `409` `"Só é possível trocar o dia até 2 horas antes do horário do agendamento"`. Nada muda. |
| **E-17** (AC-26) | `campeonato_agendamento` `CONFIRMED` a < 2 h do jogo | `PATCH /campeonato-agendamentos/:id/trocar-dia` (`x-api-key`), inclusive com `cancelar_conflitos: true` | `409` com a mesma mensagem. Nada muda. Do lado de `campeonatos` o `trocar-dia` propaga o `409` (via `handleHttpError` genérico). |
| **E-18** (AC-32) | — | `POST /api/v1/messages/send-by-key` **sem `x-api-key`** | `401 {"message": "Unauthorized"}`. Com `x-api-key` mas `key` inexistente → `404`. Sem `to` ou `key` → `400`. |
| **E-19** (auth pública) | — | Chamar `PATCH /reservas/protocol/:number/reagendar` **com** um Bearer inválido | `200`/erro de negócio normalmente — a rota é **pública**, ignora `Authorization`. |
| **E-20** (concorrência) | Reserva `waiting_approve`; 2 requisições de reagendar simultâneas para o **mesmo** slot novo | Disparar as duas quase juntas | Uma vence (slot fica `available: false`); a outra falha com `409` (`validateSchedulingsAreAvailable`) **ou** o `blockSchedulings` da 2ª não encontra o slot disponível. Nenhuma reserva fica sem `scheduling_id`. |

---

## 4. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **X-1** (AC-11) | RA `approved` **sem `cliente`** (legado) | `PATCH /ranking-agendamentos/protocolo/:n/cancelar` | `200`, cancela normalmente; **nenhuma** chamada ao `whatsapp`. |
| **X-2** (AC-12) | RA `waiting_approve` (comprovante enviado, ainda não aprovado) | Cancelar por protocolo | `200`, `cancelled`, bloqueio liberado, **sem** WhatsApp (só `approved` dispara). |
| **X-3** (AC-21) | Cenário H-7, mas `WHATSAPP_API_URL` **não configurada** em `agendamentos` | Reagendar | `200`; auditoria gravada; o adapter faz no-op silencioso (`console.warn`). |
| **X-4** (AC-21 / robustez) | Cenário H-7, `whatsapp` fora do ar (connection refused) | Reagendar | `200`; reserva re-vinculada e auditada; erro só no log de `agendamentos` (o usecase envolve a chamada em `try/catch` próprio, além do adapter). |
| **X-5** (janela — limite exato) | Reserva cujo slot mais cedo começa **exatamente** em 2 h (± segundos) | `GET /reservas/protocol/:number` e depois `cancel` | `can_cancel` reflete `now <= deadline` (limite **inclusivo**). Perto do limite pode alternar entre chamadas — comportamento esperado. |
| **X-6** (fuso -03:00) | Slot em "2026-10-20 20:00" (horário local BRT); relógio do container em UTC | `GET /reservas/protocol/:number` | `getSlotStartDate` monta `2026-10-20T20:00:00-03:00`; `deadline` = `18:00 BRT` = `21:00Z`. Conferir que a janela usa o fuso, não UTC puro. |
| **X-7** (multi-slot com 1 inválido) | Reserva de 3 slots; reagendar com 3 slots novos, sendo **1** indisponível | Reagendar | `409` **atômico** — os 3 antigos permanecem vinculados; **nenhum** slot novo é bloqueado. |
| **X-8** (reagendar para o mesmo dia/horário) | Reserva; reagendar informando **os mesmos** `{unit,court,date,start_time,end_time}` atuais | Reagendar | O `scheduling` atual está `available: false` (é da própria reserva) → `validateSchedulingsAreAvailable` falha → `409`. *(Aceitável: "trocar" para o mesmo lugar não faz sentido; se o QA achar que deveria ser no-op, registrar como melhoria futura.)* |
| **X-9** (RA reschedule muda duração) | RA de 60 min; slot novo de 90 min disponível | `PATCH /ranking-agendamentos/protocolo/:n/reagendar` com `end_time` +90 min | O RA passa a ter `hora_inicio`/`hora_fim` **exatamente** do slot resolvido (bloqueio bate com o `scheduling`). Sem extensão até o fechamento (regra do campeonato não se aplica ao ranking). |
| **X-10** (AC-30) | RA `expired` (24 h sem comprovante — 006a) numa quadra/data | Tentar criar nova reserva ou bloqueio nessa quadra/data que colidiria com o RA `expired` | **Sucesso** — o RA `expired` **não** aparece como bloqueador (`find-confirmed-ranking-...` filtra `$nin: ['cancelled','expired']`). |
| **X-11** (auditoria RA — `trocar-dia` interno) | RA `CONFIRMED` a > 2 h | `PATCH /ranking-agendamentos/:id/trocar-dia` (`x-api-key`) | Grava `ranking_agendamento_auditorias` **só** com `dia_anterior`/`dia_novo` (sem `slot_anterior`/`slot_novo` — esses são exclusivos do reagendamento público). |
| **X-12** (contato inválido no agendar-ranking) | `POST /rankings/:id/agendar` com `contato: { nome: "X" }` (sem `email`/`telefone`) | Chamada | `400` do DTO de `campeonatos` (`contato` é objeto com 3 campos obrigatórios **quando presente**). Sem `contato` → `201` normal (AC-29). |
| **X-13** (seed idempotente — AC-33) | Banco já tem `cancelamento-reembolso` com `editable_body` **editado** pelo admin (`PATCH /message-templates/:key`) | Reiniciar o serviço `beach-center-whatsapp` | O `editable_body` editado **não** é sobrescrito (`$setOnInsert`); `key`s ausentes são inseridas; log `"N inserted, M already present"`. |
| **X-14** (cancelar reserva com `motivo` > 280) | Reserva `waiting_approve` | `PATCH .../cancel` com `{ "motivo": "<300 chars>" }` | `400` — o DTO de body do cancel valida `.max(280)` **quando informado** (mas aceita ausência total). |

---

## 5. Checklist de regressão (fluxos vizinhos)

| ID | Área | Verificação |
|---|---|---|
| **R-1** | `PATCH /reservas/protocol/:number/cancel` (fluxo 006a) | Cancelamento por protocolo sem `motivo` continua funcionando (não regrediu com a adição do DTO de body). |
| **R-2** | `PATCH /reservas/:id/delete` (admin) | Cancelamento admin: `refund_status: 'manual'` se `approved`; **agora também dispara WhatsApp** `cancelamento-reembolso` quando `approved` (D3 — inclui `close-date` com `cancel_reserves: true`). Confirmar que uma falha do WhatsApp não trava o `delete`. |
| **R-3** | `GET /reservas/protocol/:number` (006a) | `can_cancel`/`cancellation_deadline` seguem corretos; `pending` agora dá `404` (antes retornava a reserva). |
| **R-4** | `POST /day/close-date` com `cancel_reserves: true` | Cancela reservas ativas (`pending`/`waiting_approve`/`approved`); as `approved` disparam WhatsApp; janela de 2 h ignorada (`skipCancellationTimeValidation`). |
| **R-5** | `PATCH /ranking-agendamentos/:id/trocar-dia` (task 005) | Troca de dia interna: conflito `409`, auditoria, `motivo` obrigatório — **+ guarda de 2 h nova**. A > 2 h, tudo igual à 005. |
| **R-6** | `PATCH /campeonato-agendamentos/:id/trocar-dia` (task 005) | Fluxo `409` + `cancelar_conflitos` preservado; **+ guarda de 2 h nova** (rejeita mesmo com `cancelar_conflitos: true`). |
| **R-7** | `POST /ranking-agendamentos` (create-bulk interno) | Aceita `cliente` opcional; sem `cliente` continua criando; `toDomainRankingAgendamento` mapeia `cliente` quando presente. |
| **R-8** | `POST /rankings/:id/agendar` (campeonatos) | `contato` opcional; sem ele, contrato inalterado; com ele, repassa `cliente` por lote. |
| **R-9** | Kernel `EventConflictService` (5 fontes) | `ranking_agendamento` `pending`/`waiting_approve`/`approved` seguem bloqueando; `expired` e `cancelled` **não** bloqueiam. Nenhuma regressão nas outras 4 fontes. |
| **R-10** | `PATCH /ranking-agendamentos/protocolo/:n/comprovante` (task 005) | Rota pública de comprovante ainda funciona; a ordem das rotas `/protocolo/...` (comprovante, GET, cancelar, reagendar) antes de `/:id` está correta (Express casa o literal primeiro). |
| **R-11** | `POST /api/v1/messages/send-text` (whatsapp, pré-existente) | Continua funcionando (`send-by-key` foi **adicionado** ao lado, não substituiu). |
| **R-12** | `GET /message-templates` / `PATCH /message-templates/:key` (whatsapp) | Após o seed, os 2 templates novos aparecem em `GET` e podem ser editados via `PATCH` (antes o `PATCH` dava `404` para `key`s só-em-memória). |
| **R-13** | `beach-center-server` | `docker-compose.dev.yml` sobe `agendamentos` com `WHATSAPP_API_URL`/`WHATSAPP_INTERNAL_API_KEY`; serviço `whatsapp` (5004) sobe e roda o seed no boot. |
| **R-14** | Mensalista / aula-bloqueio | Sem alteração — smoke test de criação/cancelamento. |

---

## 6. Rastreabilidade AC → cenário

| AC | Cenário(s) | Cobertura automatizada equivalente? |
|---|---|---|
| AC-1 — reserva antes de `waiting_approve` → 404 | E-1, E-6 | ✅ `find-by-protocol.usecase.spec`, `delete-reserve.usecase.spec` |
| AC-2 — reserva `waiting_approve` → dados + flags | H-1 | ✅ `find-by-protocol.usecase.spec` |
| AC-3 — flags falsas p/ terminal / fora da janela | E-3, X-5 | ✅ `find-by-protocol.usecase.spec` |
| AC-4 — RA `pending` → 404; `waiting_approve` → dados + flags | H-2, E-2 | ✅ `find-ranking-agendamento-by-protocol.usecase.spec` |
| AC-5 — cancelar reserva `waiting_approve` na janela | H-3 | ✅ `delete-reserve.usecase.spec` |
| AC-6 — cancelar reserva `approved` → WhatsApp reembolso | H-4, H-5 | ✅ `delete-reserve.usecase.spec` (port mockado) |
| AC-7 — falha do WhatsApp não aborta o cancelamento | H-4, X-4 | ✅ `send-whatsapp-notification.adapter.spec` + `try/catch` no usecase |
| AC-8 — cancelar fora da janela de 2 h | E-4 | ✅ `delete-reserve.usecase.spec` (herdado 006a) |
| AC-9 — cancelar reserva já terminal | E-5 | ✅ `delete-reserve.usecase.spec` |
| AC-10 — cancelar RA `approved` + `cliente.telefone` → WhatsApp | H-6 | ✅ `cancel-ranking-agendamento-by-protocol.usecase.spec` |
| AC-11 — cancelar RA `approved` sem `cliente.telefone` | X-1 | ✅ mesma spec |
| AC-12 — cancelar RA `waiting_approve` → sem WhatsApp | X-2 | ✅ mesma spec |
| AC-13 — reagendar reserva (feliz, 1 slot) | H-7 | ✅ `reschedule-reserve-by-protocol.usecase.spec` |
| AC-14 — reagendar preserva o nº de slots (2/3) | H-8 | ✅ mesma spec |
| AC-15 — nº de slots diferente → 409 | E-8 | ✅ mesma spec |
| AC-16 — slot novo inexistente / indisponível → 409 | E-9 | ✅ mesma spec |
| AC-17 — slot novo em conflito (5 fontes) → 409 | E-10 | ✅ mesma spec |
| AC-18 — slot novo no passado ou < 2 h → erro | E-11 | ✅ mesma spec |
| AC-19 — reagendar fora da janela dos slots atuais | E-12 | ✅ mesma spec |
| AC-20 — reagendar terminal → 409; `pending` → 404 | E-13 | ✅ mesma spec |
| AC-21 — falha do WhatsApp não aborta o reagendamento | X-3, X-4 | ✅ `send-whatsapp-notification.adapter.spec` + `try/catch` nos usecases |
| AC-22 — reagendar RA `waiting_approve` (feliz) | H-9 | ✅ `reschedule-ranking-agendamento-by-protocol.usecase.spec` |
| AC-23 — RA: conflito no slot novo → 409 | E-15 | ✅ mesma spec |
| AC-24 — RA: fora da janela / terminal / `pending` | E-7 | ✅ mesma spec + `find-…by-protocol.usecase.spec` |
| AC-25 — `trocar-dia` interno de RA a < 2 h → 409 | E-16 | ✅ `change-day-ranking-agendamento.usecase.spec` |
| AC-26 — `trocar-dia` interno de campeonato a < 2 h → 409 | E-17 | ✅ `change-day-campeonato-agendamento.usecase.spec` |
| AC-27 — `trocar-dia` interno a > 2 h continua funcionando | H-10, R-5, R-6 | ✅ suítes de `change-day` (fixtures +30d) |
| AC-28 — `agendar` com `contato` propaga `cliente` | H-11 | ✅ `create-bulk-ranking-agendamentos.usecase.spec` + `agendar-ranking.usecase.spec` (campeonatos) |
| AC-29 — `agendar` sem `contato` continua válido | X-12 | ✅ mesmas specs |
| AC-30 — `ranking_agendamento` `expired` não bloqueia | X-10 | ✅ `find-confirmed-ranking-agendamentos-by-court-unit.adapter.spec` |
| AC-31 — `tsc` / ESLint limpos | (build) | ✅ verificado nos 3 repos TS |
| AC-32 — `send-by-key` resolve e envia | H-12, E-18 | ⚠️ **`beach-center-whatsapp` sem suíte** (isento) — **validar manualmente** |
| AC-33 — seed idempotente | X-13 | ⚠️ **`beach-center-whatsapp` sem suíte** — **validar manualmente** |

**Cenários sem equivalente automatizado direto** (validar manualmente): H-5 (mensagem real via
Meta), H-9/H-7 com efeito de `scheduling`/bloqueio no Mongo, X-6 (fuso na fronteira dos 2 h),
X-10 (kernel com dados reais), E-16/E-17 (rotas internas com `x-api-key`), E-18/H-12/X-13
(`beach-center-whatsapp` — `send-by-key` + seed), R-2/R-4 (WhatsApp no `close-date`), E-20
(concorrência).

---

## 7. Observações para o QA

- **Não-bloqueante de verdade**: para AC-7/AC-21, a forma mais confiável de testar é derrubar o
  container `whatsapp` (ou usar credencial Meta inválida) e confirmar que a reserva/RA muda de
  estado normalmente — o erro só deve aparecer no **log de `agendamentos`**, nunca na resposta HTTP.
- **`motivo`**: cancelar aceita **sem** corpo (`{}` ou nada); reagendar **exige** `motivo`
  (`1..280` chars). Um `motivo` no cancelamento é aceito e vai para as `vars` do WhatsApp (o
  template padrão não usa `{{motivo}}`, mas o admin pode adicionar).
- **Ordem das rotas `/protocolo/...`**: `GET /protocolo/:n`, `PATCH /protocolo/:n/cancelar`,
  `PATCH /protocolo/:n/reagendar` e `PATCH /protocolo/:n/comprovante` **têm** que casar antes de
  `/:id`. Testar `GET /ranking-agendamentos/protocolo/<protocolo real>` e confirmar que **não**
  cai no `readRankingAgendamento` (`/:id`, que exigiria `x-api-key`).
- **Auditoria**: `reserva_reagendamento_auditorias` (coleção nova) só recebe registro no
  reagendamento **público** de reserva. `ranking_agendamento_auditorias` recebe tanto do
  `trocar-dia` interno (só dias) quanto do reagendamento público (dias + `slot_anterior`/`slot_novo`).
- **Slots do dia novo**: se o dia escolhido no reagendamento ainda não foi provisionado (nenhum
  `scheduling` gerado), o `findSchedulingsByExactSlots` devolve menos slots que o pedido → `409`.
  O front futuro deve carregar `GET /dias/:data/agendamentos` antes de oferecer as opções.
