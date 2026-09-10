# Plano — 006b Troca de dia e cancelamento público por protocolo

> Gerado por `/speckit-plan`. **Não implementa nada.** Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.
> Todas as regras de negócio do `context.md` estão fechadas (rodadas de perguntas + Q1–Q5).
> **Parte 2 de 2.** Depende da **task 006a** (já implementada/pushada): máquina de estados
> `pending → waiting_approve → approved`, `expired`, protocolo tardio, fim do reembolso automático.

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Serviço | Papel nesta task | Justificativa da fronteira |
|---|---|---|
| **`services/beach-center-bff-agendamentos`** | Fluxo **público por protocolo** de troca de dia (reserva + `ranking_agendamento`) e cancelamento (novo p/ ranking); `GET` público por protocolo p/ ranking; `can_reschedule`/`reschedule_deadline` no `find-by-protocol`; **janela de 2 h** no público **e** nas rotas internas de `trocar-dia` da task 005; auditoria de reagendamento de reserva (coleção nova); `ranking_agendamento` ganha `cliente`; disparo de **WhatsApp** (porta + adapter, não-bloqueante) no cancelamento e na troca. Fix de um gap da 006a (`expired` ainda contava como bloqueador no kernel). | Dono das reservas, do `ranking_agendamento`, do kernel de conflito e das rotas de `trocar-dia`. |
| **`services/beach-center-whatsapp`** | `POST /api/v1/messages/send-by-key` `{ to, key, vars }` (resolve o template por `key`, interpola, envia); **seed** dos `whatsapp_message_templates` (novos templates desta task). Serviço **JS** — **isento de Arquitetura Hexagonal e de testes unitários** (decisão do usuário, `context.md` §84–85, §209). | Já tem o esqueleto de templates (`MessageTemplateService`) + `send-text` + `WhatsAppService` (Meta Cloud API). |
| **`services/beach-center-bff-campeonatos`** | `agendar-ranking` passa `contato: { nome, email, telefone }` no `create-bulk` (Q3 — um contato por lote); o `trocar-dia-*-client.adapter` **já** propaga `4xx` via `handleHttpError` — **sem código novo** para a janela de 2 h. | É quem cria e reagenda partidas de ranking. |
| ~~`beach-center-app`~~ | **NÃO tocado.** Página `/protocolo/:number`, modal de cancelamento, aviso de reembolso = task de frontend futura. | — |
| ~~`beach-center-bff-pagamentos`~~ | **NÃO tocado** (só na 006a). | — |

### Stack

- `agendamentos` / `campeonatos`: Node + TypeScript, Express, Mongoose, `yup`, Jest + ts-jest
  (`coverageThreshold` global 80%), ESLint flat config (`exactOptionalPropertyTypes: true`,
  `strict: true`), `nanoid`.
- `beach-center-whatsapp`: Node + **JavaScript** (CommonJS), Express, Mongoose, `axios`. Sem Jest,
  sem hexagonal (exceção formalizada pelo usuário).

### Estado atual relevante (levantado no `context.md` e reconferido)

- **Janela de 2 h**: `domain/usecases/shared/reserve-cancellation-window.ts` —
  `getCancellationDeadline(schedulings: {start_time: Date}[])` pega o `start_time` do slot mais
  cedo e subtrai 2 h; `isWithinCancellationWindow(deadline)` = `now <= deadline`. Usada por
  `DeleteReserveUsecase` e `FindReserveByProtocolUsecase`.
- **Reserva pública por protocolo** (`reserva.route.ts`, **sem** auth):
  `GET /reservas/protocol/:number` → `FindReserveByProtocolUsecase` (retorna `can_cancel` +
  `cancellation_deadline`); `PATCH /reservas/protocol/:number/cancel` →
  `DeleteReserveUsecase.executeByProtocol` (após 006a: rejeita `cancelled`/`expired`; sem
  reembolso — grava `refund_status: 'manual'` quando era `approved`).
- **`scheduling_id`** da reserva é **array** (1–3). Resolver "slot escolhido → `scheduling` doc"
  já existe: `IFindSchedulingsByExactSlotsPort` (match por `date` + `start_time` + `court` +
  `unit`), usado pelo `close-date`.
- **Validação de conflito de slot** (5 fontes + dia fechado + disponibilidade): já encapsulada em
  `ReserveSchedulingValidator` (`validateSchedulingsAreNotPast`,
  `validateSchedulingsDoNotConflictWithExceptions`, `validateSchedulingsAreAvailable`). "Dia
  fechado" = `close-date` marca `available: false`, então cai no `validateSchedulingsAreAvailable`.
- **Não existe** troca de dia / reagendamento para reserva comum.
- **`ranking_agendamento`** (task 005): `RankingAgendamentoStatus` = `pending | waiting_approve |
  approved | rejected | cancelled | expired` (`+ expired` na 006a). `PATCH
  /ranking-agendamentos/:id/trocar-dia` (interna, `x-api-key`) → `ChangeDayRankingAgendamentoUsecase`
  (troca só a `data`, conflito `409` `RankingAgendamentoConflictError`, **sem janela de tempo**,
  `motivo` já obrigatório, `usuario_nome` no corpo). Não guarda `cliente`. Não há `GET` público
  nem cancelamento público. `numero_protocolo` gerado no `create-bulk`.
- **`campeonato_agendamento`**: `ChangeDayCampeonatoAgendamentoUsecase` — troca só a `data`, fluxo
  `409` `CampeonatoAgendamentoConflictError` + `cancelar_conflitos`, `motivo` obrigatório, **sem
  janela de tempo**.
- **Auditoria de troca de dia**: `ranking-agendamento-auditoria.model.ts` +
  `campeonato-agendamento-auditoria.model.ts` + coleções `*_auditorias`. Shape:
  `{ agendamento_id, id_ranking|id_campeonato, usuario_nome, motivo, dia_anterior, dia_novo,
  created_at }`.
- **`find-confirmed-ranking-agendamentos-by-court-unit.adapter.ts`** (5ª fonte do kernel):
  filtra `status: { $ne: "cancelled" }`. **BUG latente da 006a**: um `ranking_agendamento`
  `expired` **ainda** conta como bloqueador (a 006a adicionou `expired` ao enum + libera o
  `scheduling`, mas não excluiu `expired` desta consulta).
- **`beach-center-whatsapp`**: `MessageTemplateService.get(key, vars)` já resolve template do banco
  (`whatsapp_message_templates`) com fallback para `src/domain/default-message-templates.js`, e
  interpola `{{var}}`. `POST /api/v1/messages/send-text` `{ to, text }` (`x-api-key` =
  `WHATSAPP_INTERNAL_API_KEY`) → `WhatsAppService.sendTextMessage`. **Não há** rota `send-by-key`
  nem seed no banco (só o fallback em memória). `PATCH /message-templates/:key` exige a linha
  existir no banco (`404` senão) — por isso o seed é necessário para o admin poder editar.
- **`campeonatos`**: `AgendarRankingUsecase` chama `createBulkRankingAgendamentosClientPort.execute
  ({ id_ranking, quadras, datas, hora_inicio, duracao_partida_minutos, modalidade, quantidade })`
  — **não passa dados de cliente**. `agendarRankingDTO` = `{ quadras, datas, hora_inicio,
  duracao_partida_minutos }`.
- **`agendamentos/config/env.ts`**: **não tem** `WHATSAPP_API_URL` / `WHATSAPP_INTERNAL_API_KEY`.

## Decisões de design desta etapa

**DD1 — `motivo` obrigatório e curto.** Nas ações **públicas** (cancelar e reagendar) o cliente
informa `motivo` (string). Validação: `trim()` não-vazio, `≤ 280` chars. Gravado na auditoria de
reagendamento; no cancelamento não há auditoria (mantém o comportamento atual) — o `motivo` é só
registrado no log e enviado ao WhatsApp como `{{motivo}}` se o template usar.

**DD2 — Janela de 2 h reutilizável.** Novo `domain/usecases/shared/agendamento-time-window.ts`:
- `getSlotStartDate(dateKey: string, timeHHmm: string): Date` — `parseLocalDate` + horas/min.
- `getWindowDeadline(slotStart: Date, hours = 2): Date` — `slotStart − hours`.
- `isWithinWindow(deadline: Date): boolean` — `Date.now() <= deadline.getTime()`.
- `getEarliestSlotStart(slots: { start_time: Date }[]): Date | null` — para a reserva (array).
`reserve-cancellation-window.ts` **passa a delegar** para este módulo (mantém `getCancellationDeadline`
/ `isWithinCancellationWindow` como wrappers — zero mudança nos call sites atuais).

**DD3 — Reagendamento resolve slots EXISTENTES, não cria `scheduling`.** O cliente escolhe
`{ unit, court, date, start_time, end_time }[]` (1..3, mesmo nº do original). O usecase resolve
cada escolha via `IFindSchedulingsByExactSlotsPort` (match por `date`+`start_time`+`court`+`unit`).
Se algum não casar com um `scheduling` existente → **409** (`"Horário indisponível para
reagendamento"`). Os slots do dia são provisionados pelo admin (`create-day` /
`list-day-schedulings`) — o front futuro carrega o dia antes de oferecer as opções. *(Alternativa
rejeitada: criar `scheduling` docs a partir de rota pública — abre superfície de escrita não
controlada e duplica a lógica de `create-day`.)*

**DD4 — Validação por slot novo (reagendamento).** Para o conjunto de `scheduling_id` novos:
`ReserveSchedulingValidator.validateSchedulingLimit` (nº = nº atual, `≤ 3`, sem repetição) +
`validateSchedulingsAreNotPast` + `validateSchedulingsDoNotConflictWithExceptions` (5 fontes) +
`validateSchedulingsAreAvailable` (cobre "dia fechado"). **Mais**: cada slot novo também respeita
a janela de 2 h (`getSlotStartDate(new.date, new.start_time)` — não pode começar a menos de 2 h).
E a janela de 2 h sobre os slots **atuais** (o reagendamento em si só é permitido até 2 h antes do
jogo atual).

**DD5 — Ordem transacional do reagendamento.**
1. Busca por protocolo → `404` se não achar; `409`/`404` se `status === 'pending'` (sem protocolo
   ativo — DD8); `409` se `status` terminal (`cancelled`/`rejected`/`expired`).
2. Janela de 2 h sobre os slots atuais (`409` `"Fora da janela de 2 horas"`).
3. Resolve + valida os slots novos (DD3/DD4).
4. **Bloqueia** os `scheduling` novos (`schedulingAvailabilityService.blockSchedulings`).
5. **Re-vincula** o registro: reserva → `ISetReserveSchedulingsPort` (novo `scheduling_id`); RA →
   `IRescheduleRankingAgendamentoPort` (`court`/`unit`/`data`/`hora_inicio`/`hora_fim`).
6. **Libera** os `scheduling` antigos (`releaseSchedulingsIfAllowed` — só solta o que nada mais
   bloqueia). Para RA: `eventSchedulingImpactService.releaseEventFromSchedulings` (janela antiga)
   + `applyEventToSchedulings` (janela nova).
7. Grava auditoria (DD16).
8. Dispara WhatsApp `reagendamento-confirmado` (**não-bloqueante** — DD10).
`number`/`numero_protocolo` e `total`/preço **intactos** (nº de slots preservado).

**DD6 — Reagendamento de `ranking_agendamento` = 1 slot.** Um RA = 1 partida = 1 quadra = 1 data.
O reschedule refaz `unit`/`court`/`data`/`hora_inicio`/`hora_fim` (a duração pode mudar se o
cliente escolher outro horário — mantém `hora_fim = start + (hora_fim_atual − hora_inicio_atual)`?
**Não** — usa exatamente o `start_time`/`end_time` do `scheduling` resolvido, para bater com o
bloqueio). Novo port `IRescheduleRankingAgendamentoPort` + adapter `ranking_agendamento/reschedule/`.

**DD7 — `ranking_agendamento.cliente`.** `IRankingAgendamento` ganha
`cliente?: { nome: string; email: string; telefone: string }` (**opcional** — RAs legados não
têm). Model + schema (subdocumento `{ _id: false }`) + `ICreateRankingAgendamentoData` +
`create-bulk` DTO/usecase/input-port + `create-many` adapter. Origem: campo `contato` no
`POST /rankings/:id/agendar` do MS de campeonatos (Q3 — um `contato` para o lote inteiro,
repassado igual em todas as partidas).

**DD8 — `GET` público por protocolo: 404 antes de `waiting_approve`.** Regra B-btn: as ações
públicas existem a partir de `waiting_approve`.
- **Reserva**: `pending` não tem `number` → `find-by-protocol` já devolve `null` → `404`. Guarda
  extra: se achar por `number` mas `status === 'pending'`, devolve `null` (defensivo).
- **`ranking_agendamento`**: `numero_protocolo` existe desde o `create-bulk`; a busca pública
  devolve **`404`** quando `status === 'pending'`.

**DD9 — `can_cancel` / `can_reschedule` + deadlines.** Ambos calculados da mesma janela de 2 h
sobre os slots **atuais**.
- `can_cancel = false` para `cancelled` / `rejected` / `expired` ou sem slots ou fora da janela.
  *(006a já cobre `cancelled`/`expired`; adicionar `rejected`.)*
- `can_reschedule` = mesma regra de `can_cancel`.
- `cancellation_deadline` / `reschedule_deadline` = o mesmo `Date` (deadline da janela) — expostos
  mesmo quando já passou (o front mostra "expirou").
- `IReserve` ganha `can_reschedule?: boolean` e `reschedule_deadline?: Date | string` (segue o
  padrão já existente de `can_cancel`/`cancellation_deadline` no model).

**DD10 — WhatsApp em `agendamentos` (porta de saída, não-bloqueante).**
- `domain/ports/output/whatsapp-notification.port.ts` **(NOVO)** — `ISendWhatsappNotificationPort`
  `execute(input: { to: string; key: string; vars?: Record<string, string> }): Promise<void>`.
- `infra/adapters/whatsapp/send-whatsapp-notification.adapter.ts` **(NOVO)** —
  `POST {WHATSAPP_API_URL}/messages/send-by-key` com `x-api-key: WHATSAPP_INTERNAL_API_KEY`.
  **Nunca lança**: try/catch interno → `console.error` e retorna. Se `WHATSAPP_API_URL` não estiver
  configurada, no-op silencioso (log em nível `warn`).
- Os usecases chamam o port com `await` mas o adapter engole o erro — o cancelamento/troca
  **sempre** conclui.

**DD11 — Gatilhos de WhatsApp** (só nos fluxos **públicos**):
| Fluxo | Condição | Template `key` | `to` | `vars` |
|---|---|---|---|---|
| Cancelar reserva (`DeleteReserveUsecase`, D10) | `reserve.status === 'approved'` no momento do cancelamento | `cancelamento-reembolso` | `reserve.phone` | `{ protocolo, nome }` |
| Cancelar RA por protocolo | `status === 'approved'` **e** `cliente?.telefone` (Q2) | `cancelamento-reembolso` | `cliente.telefone` | `{ protocolo, nome }` |
| Reagendar reserva | sucesso | `reagendamento-confirmado` | `reserve.phone` | `{ protocolo, nome, novo_dia, novo_horario, quadra }` |
| Reagendar RA | sucesso, se `cliente?.telefone` | `reagendamento-confirmado` | `cliente.telefone` | idem |
As rotas **internas** de `trocar-dia` (task 005) **não** disparam WhatsApp.

**DD12 — `beach-center-whatsapp`: `send-by-key` + seed.**
- `applications/routes/message.route.js` **(ALT)** — `+ POST /send-by-key`.
- `applications/controllers/messages/send-by-key.controller.js` **(NOVO)** — mesma guarda
  `x-api-key` do `send-text`; `{ to, key, vars }` (400 se faltar `to`/`key`) →
  `new MessageTemplateService().get(key, vars || {})` → `new WhatsAppService().sendTextMessage(to,
  body)`. `404` se o template resolver para string vazia (`key` inexistente).
- `domain/default-message-templates.js` **(ALT)** — `+` `cancelamento-reembolso` e
  `reagendamento-confirmado` (PT-BR, `editable: true`). Opcional: placeholders inertes do bot
  futuro (`boas-vindas`, `day-use`, `agendamento-quadra`, `pagamento-confirmado`) como registros
  iniciais.
- `infra/seed/seed-message-templates.js` **(NOVO)** + chamada em `index.js` (após `mongoose.connect`)
  — **upsert idempotente**: para cada entry de `default-message-templates.js`, `updateOne({ key },
  { $setOnInsert: {...} }, { upsert: true })` — **nunca** sobrescreve `editable_body` que o admin
  já mexeu. Falha do seed não derruba o boot (try/catch + log).
Textos (Q5 — sem revisão; `editable`):
- `cancelamento-reembolso` — `editable_body`: *"Olá, {{nome}}! Sua reserva {{protocolo}} foi
  cancelada. Como havíamos recebido o pagamento, o valor será tratado por aqui mesmo — **responda
  esta conversa** e nossa equipe cuida do reembolso com você."* / `fixed_body`: `""`.
- `reagendamento-confirmado` — `editable_body`: *"Prontinho, {{nome}}! Sua reserva {{protocolo}}
  foi remarcada para **{{novo_dia}}** às **{{novo_horario}}** na {{quadra}}. Nos vemos lá!"* /
  `fixed_body`: `""`.

**DD13 — Janela de 2 h nas rotas internas de `trocar-dia` (task 005).**
- `ChangeDayCampeonatoAgendamentoUsecase` e `ChangeDayRankingAgendamentoUsecase` **(ALT)** — logo
  após carregar `existing` e validar `motivo`/status: `const slotStart =
  getSlotStartDate(existing.data, existing.hora_inicio)`; se
  `!isWithinWindow(getWindowDeadline(slotStart, 2))` → lançar
  `new ConflictError("Só é possível trocar o dia até 2 horas antes do horário do agendamento")`
  (**409**). *(Reusa `ConflictError` — o `trocar-dia-*-client.adapter` de `campeonatos` já
  propaga `409` via `handleHttpError`; nenhum código novo do lado de campeonatos.)*
- Confirmar no `/speckit-implement` que o `handleHttpError` de `campeonatos` repassa o `message`
  do `409` sem mascarar.

**DD14 — `campeonatos`: `contato` no `agendar-ranking`.**
- `applications/dto/ranking-agendar.dto.ts` **(ALT)** — `+ contato: yup.object({ nome, email,
  telefone }).optional()` (ou `.default(undefined)`).
- `domain/ports/input/ranking.input-port.ts` **(ALT)** — `IAgendarRankingData + contato?`.
- `domain/usecases/ranking/agendar/agendar-ranking.usecase.ts` **(ALT)** — repassa `cliente:
  data.contato` no `createBulkRankingAgendamentosClientPort.execute`.
- `domain/ports/output/ranking-agendamento-client.port.ts` **(ALT)** —
  `ICreateBulkRankingAgendamentosClientPayload + cliente?`.
- `infra/adapters/ranking-agendamento-client/create-bulk/*.adapter.ts` **(ALT)** — inclui
  `cliente` no body do `POST`.

**DD15 — Fix do gap da 006a (`expired` no kernel de ranking).**
- `infra/adapters/ranking_agendamento/find-confirmed-by-court-unit/find-confirmed-ranking-agendamentos-by-court-unit.adapter.ts`
  **(ALT)** — `status: { $ne: "cancelled" }` → `status: { $nin: ["cancelled", "expired"] }`.
- `campeonato_agendamento` não tem `expired` → sem mudança. `has-active-reserve` /
  `find-active-reserves` da reserva já estão corretos (006a).

**DD16 — Auditoria de reagendamento.**
- **Reserva** — coleção **nova** `reserva_reagendamento_auditorias`:
  `domain/models/reserva-reagendamento-auditoria.model.ts` **(NOVO)** —
  `{ id, reserva_id, numero_protocolo, motivo, slots_anteriores: ISlotSnapshot[], slots_novos:
  ISlotSnapshot[], created_at }` onde `ISlotSnapshot = { scheduling_id, date, start_time,
  end_time, court, unit }`.
- **`ranking_agendamento`** — **reusa** `ranking_agendamento_auditorias`. Estende
  `IRankingAgendamentoAuditoria` / `ICreateRankingAgendamentoAuditoriaData` **(ALT)** com
  `slot_anterior?: ISlotSnapshot` e `slot_novo?: ISlotSnapshot` (opcionais — o `trocar-dia`
  interno continua gravando só `dia_anterior`/`dia_novo`). No reschedule público: `usuario_nome =
  existing.cliente?.nome ?? "Cliente (protocolo)"`.

**DD17 — Autorização.** As 5 rotas públicas novas (`GET`/`PATCH` por protocolo de reserva e
ranking) **sem** middleware, padrão de `reserva.route.ts`. `POST /messages/send-by-key` com
`x-api-key`. As rotas internas de `trocar-dia` seguem `x-api-key`.

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras** | 3 repos: `agendamentos`, `beach-center-whatsapp`, `beach-center-bff-campeonatos` | **CONFORME.** `agendamentos` é dono do estado das reservas/RA, do kernel e das rotas de `trocar-dia`; fala com `whatsapp` **só** por uma porta de saída (`POST /messages/send-by-key`, `x-api-key`), nunca o contrário. `campeonatos` continua sendo o único que dispara agendamento de ranking e só passa a enviar mais um campo (`contato`). `beach-center-app` e `pagamentos` intocados. Nenhum uso de pattern BFF. |
| **II — Hexagonal / Ports** | `agendamentos` e `campeonatos` hexagonais; `beach-center-whatsapp` **isento por decisão do usuário** (`context.md` §84–85, JS puro, serviço de mensageria legado) | **CONFORME.** Em `agendamentos`: novos usecases de domínio (`reschedule-by-protocol`, `cancel-by-protocol` de ranking, `find-by-protocol` de ranking), regra de janela num módulo `shared/` puro, WhatsApp atrás de `ISendWhatsappNotificationPort` (um adapter por ação), auditoria com um adapter por verbo. Nenhum adapter com regra de negócio. `beach-center-whatsapp` permanece JS não-hexagonal — **exceção pré-aprovada e documentada**, não uma violação nova. |
| **III — Test-First / Qualidade** | `coverageThreshold` global 80% em `agendamentos` e `campeonatos`; `beach-center-whatsapp` **sem suíte** (decisão do usuário) | **CONFORME** (cobrado no `/speckit-unit-tests`). Specs novos: `reschedule-reserve-by-protocol` e `reschedule-ranking-agendamento-by-protocol` (status ≥ `waiting_approve`, janela 2 h nos slots atuais **e** novos, contagem de slots, conflito das 5 fontes + dia fechado + disponibilidade, release+block de `scheduling`, auditoria, `number`/`total` preservados); `cancel-ranking-agendamento-by-protocol`; `find-ranking-agendamento-by-protocol` (404 em `pending`); `find-by-protocol` de reserva (`+ can_reschedule`, 404 em `pending`); `change-day` campeonato/ranking com a guarda de 2 h; `delete-reserve` (D10 — WhatsApp mockado quando `approved`, não-bloqueante em falha); `agendamento-time-window`; `find-confirmed-ranking-...` (exclui `expired`); adapter de WhatsApp (não lança em erro). `campeonatos`: `agendar-ranking` com `contato`. ESLint estrito. **`beach-center-whatsapp` sem testes** — exceção documentada. |
| **IV — Infra hot-reload** | N/A | **CONFORME.** `agendamentos` ganha 2 env vars (`WHATSAPP_API_URL`, `WHATSAPP_INTERNAL_API_KEY`) — refletir em `beach-center-server/{docker-compose.dev.yml,.env.dev.example,README.md}`. `beach-center-whatsapp` já roda no compose; nenhum volume/serviço novo. |

**Resultado: sem violação. A não-conformidade de `beach-center-whatsapp` com II/III é uma exceção
pré-aprovada pelo usuário (registrada no `context.md`), não um ERRO desta task.**

## Mapa Arquitetural (Hexagonal)

### `services/beach-center-bff-agendamentos/src/`

```
domain/models/
  reserva.model.ts                                   (ALT)  # IReserve + can_reschedule?/reschedule_deadline?
  ranking-agendamento.model.ts                       (ALT)  # + cliente?: { nome; email; telefone }; ICreateRankingAgendamentoData + cliente?
  ranking-agendamento-auditoria.model.ts             (ALT)  # + slot_anterior?/slot_novo?: ISlotSnapshot (em ICreate* também)
  reserva-reagendamento-auditoria.model.ts           (NEW)  # IReservaReagendamentoAuditoria + ISlotSnapshot + ICreate*

domain/ports/input/
  reserva.input-port.ts                              (ALT)  # + IRescheduleReserveByProtocolUseCase (+ tipo do resultado do find-by-protocol com can_reschedule)
  ranking-agendamento.input-port.ts                  (ALT)  # + IFindRankingAgendamentoByProtocolUseCase, ICancelRankingAgendamentoByProtocolUseCase, IRescheduleRankingAgendamentoByProtocolUseCase; ICreateBulkRankingAgendamentosData + cliente?

domain/ports/output/
  reserve-persistence.port.ts                        (ALT)  # + ISetReserveSchedulingsPort (re-vincula scheduling_id)
  ranking-agendamento-persistence.port.ts            (ALT)  # + IRescheduleRankingAgendamentoPort; ISetRankingAgendamentoStatusPort já existe (cancelamento)
  ranking-agendamento-auditoria-persistence.port.ts  (ALT)  # ICreate* aceita os campos de slot
  reserva-reagendamento-auditoria-persistence.port.ts (NEW) # ICreateReservaReagendamentoAuditoriaPort + IList*
  whatsapp-notification.port.ts                      (NEW)  # ISendWhatsappNotificationPort

domain/usecases/shared/
  agendamento-time-window.ts                         (NEW)  # getSlotStartDate / getWindowDeadline / isWithinWindow / getEarliestSlotStart
  reserve-cancellation-window.ts                     (ALT)  # passa a delegar para agendamento-time-window (API pública inalterada)

domain/usecases/reserva/
  find-by-protocol/find-by-protocol.usecase.ts       (ALT)  # + can_reschedule / reschedule_deadline; 404 quando status === 'pending'
  delete/delete-reserve.usecase.ts                   (ALT)  # executeByProtocol: exige status >= waiting_approve? não — mantém; + dispara WhatsApp cancelamento-reembolso quando status era 'approved' (D10); recebe ISendWhatsappNotificationPort
  reschedule-by-protocol/reschedule-reserve-by-protocol.usecase.ts  (NEW)

domain/usecases/ranking-agendamento/
  create-bulk/create-bulk-ranking-agendamentos.usecase.ts  (ALT)  # persiste `cliente`
  change-day/change-day-ranking-agendamento.usecase.ts     (ALT)  # + guarda de 2 h (DD13)
  find-by-protocol/find-ranking-agendamento-by-protocol.usecase.ts   (NEW)  # público; 404 em 'pending'; can_cancel/can_reschedule + deadlines
  cancel-by-protocol/cancel-ranking-agendamento-by-protocol.usecase.ts (NEW)
  reschedule-by-protocol/reschedule-ranking-agendamento-by-protocol.usecase.ts (NEW)

domain/usecases/campeonato-agendamento/
  change-day/change-day-campeonato-agendamento.usecase.ts  (ALT)  # + guarda de 2 h (DD13)

infra/schemas/
  ranking-agendamento.schema.ts                      (ALT)  # subdocumento `cliente`
  ranking-agendamento-auditoria.schema.ts            (ALT)  # `slot_anterior`/`slot_novo` (Mixed / subdoc)
  reserva-reagendamento-auditoria.schema.ts          (NEW)  # coleção `reserva_reagendamento_auditorias`

infra/adapters/
  reserva/set-schedulings/set-reserve-schedulings.adapter.ts                       (NEW)
  reserva_reagendamento_auditoria/{create,list}/*.adapter.ts                       (NEW)
  ranking_agendamento/reschedule/reschedule-ranking-agendamento.adapter.ts         (NEW)
  ranking_agendamento/find-by-protocol/find-ranking-agendamento-by-protocol.adapter.ts  (existe — reusa)
  ranking_agendamento/create-many/create-many-ranking-agendamentos.adapter.ts      (ALT)  # persiste `cliente`
  ranking_agendamento_auditoria/create/create-ranking-agendamento-auditoria.adapter.ts  (ALT)  # grava slot_anterior/slot_novo
  ranking_agendamento/find-confirmed-by-court-unit/*.adapter.ts                    (ALT)  # $nin ['cancelled','expired'] (DD15)
  whatsapp/send-whatsapp-notification.adapter.ts                                   (NEW)

applications/dto/
  reschedule-by-protocol.dto.ts                      (NEW)  # { number, motivo, slots: [{ unit, court, date, start_time, end_time }] } (1..3)
  ranking-reschedule-by-protocol.dto.ts              (NEW)  # { numero_protocolo, motivo, unit, court, date, start_time, end_time }
  cancel-by-protocol.dto.ts                          (NEW)  # { motivo } no body (path já validado por find-by-protocol.dto)  — ou estende
  ranking-agendamento-bulk.dto.ts                    (ALT)  # + cliente? { nome, email, telefone }

applications/controllers/
  reserva/reschedule-by-protocol/reschedule-reserve-by-protocol.controller.ts      (NEW)  # público
  reserva/cancel-by-protocol/cancel-reserve-by-protocol.controller.ts              (ALT)  # + captura `motivo` do body, repassa ao usecase
  ranking_agendamento/find-by-protocol/find-ranking-agendamento-by-protocol.controller.ts  (NEW)
  ranking_agendamento/cancel-by-protocol/cancel-ranking-agendamento-by-protocol.controller.ts (NEW)
  ranking_agendamento/reschedule-by-protocol/reschedule-ranking-agendamento-by-protocol.controller.ts (NEW)

applications/routes/
  reserva.route.ts                                   (ALT)  # + PATCH /reservas/protocol/:number/reagendar (público)
  ranking-agendamento.route.ts                       (ALT)  # + GET /ranking-agendamentos/protocolo/:numero_protocolo (público)
                                                            # + PATCH /ranking-agendamentos/protocolo/:numero_protocolo/cancelar (público)
                                                            # + PATCH /ranking-agendamentos/protocolo/:numero_protocolo/reagendar (público)
                                                            #   (ordem: literais antes de /:id — já é o padrão do arquivo)

config/
  env.ts                                             (ALT)  # + WHATSAPP_API_URL, WHATSAPP_INTERNAL_API_KEY (opcionais)
  env.spec.ts                                        (ALT)
  container.ts                                       (ALT)  # wiring: time-window (puro, sem wiring), whatsapp adapter+port, novos usecases/adapters, DeleteReserveUsecase ganha o port de WhatsApp
  container.spec.ts                                  (ALT)
```

### `services/beach-center-bff-campeonatos/src/`

```
applications/dto/ranking-agendar.dto.ts              (ALT)  # + contato? { nome, email, telefone }
domain/ports/input/ranking.input-port.ts             (ALT)  # IAgendarRankingData + contato?
domain/usecases/ranking/agendar/agendar-ranking.usecase.ts  (ALT)  # repassa cliente: data.contato
domain/ports/output/ranking-agendamento-client.port.ts     (ALT)  # ICreateBulkRankingAgendamentosClientPayload + cliente?
infra/adapters/ranking-agendamento-client/create-bulk/create-bulk-ranking-agendamentos-client.adapter.ts  (ALT)  # inclui cliente no body
infra/adapters/{campeonato,ranking}-agendamento-client/trocar-dia/*.adapter.ts  (—)  # sem mudança — handleHttpError já propaga o 409 (confirmar no implement)
```

### `services/beach-center-whatsapp/src/`

```
applications/routes/message.route.js                 (ALT)  # + POST /send-by-key
applications/controllers/messages/send-by-key.controller.js  (NEW)
domain/default-message-templates.js                  (ALT)  # + cancelamento-reembolso, reagendamento-confirmado (+ placeholders do bot futuro, opcional)
infra/seed/seed-message-templates.js                 (NEW)  # upsert idempotente ($setOnInsert)
index.js                                             (ALT)  # chama o seed após mongoose.connect (try/catch, não derruba o boot)
```

### `beach-center-server/`

```
docker-compose.dev.yml   (ALT)  # + WHATSAPP_API_URL / WHATSAPP_INTERNAL_API_KEY no serviço `agendamentos`
.env.dev.example         (ALT)  # idem (comentado)
README.md                (ALT)  # tabela de env, se listar as do agendamentos
```

## Checklist de Implementação

> Ordem por dependência: fix da 006a → helper de janela → `cliente` do ranking → WhatsApp
> (whatsapp → agendamentos) → guarda de 2 h interna → fluxos públicos de reserva → fluxos públicos
> de ranking → wiring/env → verificação. Nenhum commit (`/speckit-implement`).

### Fase 0 — Fix do gap da 006a

- [x] `services/beach-center-bff-agendamentos/src/infra/adapters/ranking_agendamento/find-confirmed-by-court-unit/find-confirmed-ranking-agendamentos-by-court-unit.adapter.ts`
      — `status: { $ne: "cancelled" }` → `status: { $nin: ["cancelled", "expired"] }`; atualizar o
      doc-comment.

### Fase 1 — Janela de 2 h reutilizável

- [x] `.../domain/usecases/shared/agendamento-time-window.ts` **(NEW)** — `getSlotStartDate`,
      `getWindowDeadline`, `isWithinWindow`, `getEarliestSlotStart` (usa `parseLocalDate` de
      `shared/date-time`).
- [x] `.../domain/usecases/shared/reserve-cancellation-window.ts` **(ALT)** — reimplementa
      `getCancellationDeadline`/`isWithinCancellationWindow` como wrappers de
      `agendamento-time-window` (assinaturas e comportamento idênticos).

### Fase 2 — `ranking_agendamento.cliente` + `contato` no `agendar-ranking` (campeonatos)

- [x] `.../agendamentos/src/domain/models/ranking-agendamento.model.ts` — `IRankingAgendamento +
      cliente?: { nome: string; email: string; telefone: string }`; `ICreateRankingAgendamentoData
      + cliente?`.
- [x] `.../agendamentos/src/infra/schemas/ranking-agendamento.schema.ts` — subdocumento `cliente`
      (`new mongoose.Schema({ nome, email, telefone }, { _id: false })`, `required: false`);
      `toDomainRankingAgendamento` mapeia `cliente` quando presente.
- [x] `.../agendamentos/src/domain/ports/input/ranking-agendamento.input-port.ts` —
      `ICreateBulkRankingAgendamentosData + cliente?`.
- [x] `.../agendamentos/src/applications/dto/ranking-agendamento-bulk.dto.ts` — `+ cliente:
      yup.object({ nome: string().required(), email: string().email().required(), telefone:
      string().required() }).optional()`.
- [x] `.../agendamentos/src/domain/usecases/ranking-agendamento/create-bulk/create-bulk-ranking-agendamentos.usecase.ts`
      — inclui `...(data.cliente ? { cliente: data.cliente } : {})` em cada `ICreateRankingAgendamentoData`.
- [x] `.../agendamentos/src/infra/adapters/ranking_agendamento/create-many/create-many-ranking-agendamentos.adapter.ts`
      — persiste `cliente` (spread condicional).
- [x] `.../campeonatos/src/applications/dto/ranking-agendar.dto.ts` — `+ contato?`.
- [x] `.../campeonatos/src/domain/ports/input/ranking.input-port.ts` — `IAgendarRankingData + contato?`.
- [x] `.../campeonatos/src/domain/usecases/ranking/agendar/agendar-ranking.usecase.ts` — repassa
      `...(data.contato ? { cliente: data.contato } : {})`.
- [x] `.../campeonatos/src/domain/ports/output/ranking-agendamento-client.port.ts` —
      `ICreateBulkRankingAgendamentosClientPayload + cliente?`.
- [x] `.../campeonatos/src/infra/adapters/ranking-agendamento-client/create-bulk/create-bulk-ranking-agendamentos-client.adapter.ts`
      — inclui `cliente` no body do `axios.post`.

### Fase 3 — `beach-center-whatsapp`: `send-by-key` + seed

- [x] `.../whatsapp/src/domain/default-message-templates.js` — `+ cancelamento-reembolso`,
      `+ reagendamento-confirmado` (textos de DD12); `variables` preenchido; `editable: true`.
- [x] `.../whatsapp/src/applications/controllers/messages/send-by-key.controller.js` **(NEW)** —
      guarda `x-api-key`; valida `to`/`key`; `MessageTemplateService.get(key, vars || {})`;
      `404` se vazio; `WhatsAppService.sendTextMessage(to, body)`; `200` com `{ data: result }`.
- [x] `.../whatsapp/src/applications/routes/message.route.js` — `+ messageRoute.post('/send-by-key',
      ...)`.
- [x] `.../whatsapp/src/infra/seed/seed-message-templates.js` **(NEW)** — itera
      `default-message-templates`, `MessageTemplateModel.updateOne({ key }, { $setOnInsert: {
      ...entry, body: composeBody(entry) } }, { upsert: true })`.
- [x] `.../whatsapp/index.js` — após `mongoose.connect(...)`, `await seedMessageTemplates()` dentro
      de try/catch (log + segue).

### Fase 4 — WhatsApp adapter/port em `agendamentos`

- [x] `.../agendamentos/src/domain/ports/output/whatsapp-notification.port.ts` **(NEW)**.
- [x] `.../agendamentos/src/infra/adapters/whatsapp/send-whatsapp-notification.adapter.ts` **(NEW)**
      — `axios.post(\`${env.WHATSAPP_API_URL}/messages/send-by-key\`, { to, key, vars }, { headers:
      { 'x-api-key': env.WHATSAPP_INTERNAL_API_KEY } })`; try/catch total (nunca lança); no-op +
      `console.warn` se `WHATSAPP_API_URL` ausente.
- [x] `.../agendamentos/src/config/env.ts` — `+ WHATSAPP_API_URL?`, `+ WHATSAPP_INTERNAL_API_KEY?`
      (opcionais, spread condicional). `env.spec.ts` acompanha.

### Fase 5 — Guarda de 2 h nas rotas internas de `trocar-dia`

- [x] `.../agendamentos/src/domain/usecases/ranking-agendamento/change-day/change-day-ranking-agendamento.usecase.ts`
      — após validar `motivo`/status: `if (!isWithinWindow(getWindowDeadline(getSlotStartDate(
      existing.data, existing.hora_inicio), 2))) throw new ConflictError("Só é possível trocar o
      dia até 2 horas antes do horário do agendamento");`.
- [x] `.../agendamentos/src/domain/usecases/campeonato-agendamento/change-day/change-day-campeonato-agendamento.usecase.ts`
      — idem (antes do `detectConflicts`).

### Fase 6 — Fluxos públicos de **reserva**

- [x] `.../domain/models/reserva.model.ts` — `IReserve + can_reschedule?: boolean;
      reschedule_deadline?: Date | string | undefined`.
- [x] `.../domain/models/reserva-reagendamento-auditoria.model.ts` **(NEW)** —
      `ISlotSnapshot`, `IReservaReagendamentoAuditoria`, `ICreateReservaReagendamentoAuditoriaData`.
- [x] `.../domain/ports/output/reserva-reagendamento-auditoria-persistence.port.ts` **(NEW)** —
      `ICreateReservaReagendamentoAuditoriaPort` + `IListReservaReagendamentoAuditoriaPort`.
- [x] `.../domain/ports/output/reserve-persistence.port.ts` — `+ ISetReserveSchedulingsPort`.
- [x] `.../domain/ports/output/whatsapp-notification.port.ts` (já da Fase 4).
- [x] `.../domain/ports/input/reserva.input-port.ts` — `+ IRescheduleReserveByProtocolData` (`{
      number, motivo, slots: { unit; court; date; start_time; end_time }[] }`) + `IRescheduleReserveByProtocolUseCase`;
      resultado do `find-by-protocol` documenta `can_reschedule`/`reschedule_deadline`.
- [x] `.../infra/schemas/reserva-reagendamento-auditoria.schema.ts` **(NEW)** — coleção
      `reserva_reagendamento_auditorias`, `timestamps`.
- [x] `.../infra/adapters/reserva_reagendamento_auditoria/{create,list}/*.adapter.ts` **(NEW)**.
- [x] `.../infra/adapters/reserva/set-schedulings/set-reserve-schedulings.adapter.ts` **(NEW)** —
      `ReserveModel.findByIdAndUpdate(id, { scheduling_id: ids.map(ObjectId) }, { new: true })` →
      `toDomainReserve`.
- [x] `.../domain/usecases/reserva/find-by-protocol/find-by-protocol.usecase.ts` — `getCancellationInfo`
      passa a devolver também `can_reschedule`/`reschedule_deadline` (mesma janela); `rejected`
      entra no conjunto que zera as flags; **retorna `null` se `status === 'pending'`**.
- [x] `.../domain/usecases/reserva/reschedule-by-protocol/reschedule-reserve-by-protocol.usecase.ts`
      **(NEW)** — construtor: `IFindReserveByProtocolPort`, `IFindSchedulingsByIdsPort`,
      `IFindSchedulingsByExactSlotsPort`, `ReserveSchedulingValidator`, `SchedulingAvailabilityService`,
      `ISetReserveSchedulingsPort`, `ICreateReservaReagendamentoAuditoriaPort`,
      `ISendWhatsappNotificationPort`. Fluxo = DD5. Erros: `404` (não achou / `pending`), `409`
      (terminal / fora da janela / nº de slots diferente / slot novo indisponível / conflito).
- [x] `.../domain/usecases/reserva/delete/delete-reserve.usecase.ts` — construtor `+
      ISendWhatsappNotificationPort`; em `execute`, **antes** de mudar o status, captura
      `wasApproved = reserve.status === 'approved'`; após o cancelamento bem-sucedido, se
      `wasApproved` → `await this.sendWhatsappNotificationPort.execute({ to: reserve.phone, key:
      'cancelamento-reembolso', vars: { protocolo: reserve.number ?? '', nome: reserve.name } })`
      (adapter é não-bloqueante). Vale para `execute` e `executeByProtocol`.
- [x] `.../applications/dto/reschedule-by-protocol.dto.ts` **(NEW)** — `number` (path),
      `motivo` (body, `≤ 280`, required), `slots` (body, array 1..3 de `{ unit, court (ObjectId),
      date (YYYY-MM-DD), start_time (HH:MM), end_time (HH:MM) }`).
- [x] `.../applications/dto/cancel-by-protocol.dto.ts` **(NEW)** — `motivo` no body (required,
      `≤ 280`). *(o `number` do path continua via `find-by-protocol.dto.ts`.)*
- [x] `.../applications/controllers/reserva/reschedule-by-protocol/reschedule-reserve-by-protocol.controller.ts`
      **(NEW)** — público; valida DTO; `container.rescheduleReserveByProtocol.execute(...)`;
      `404`/`409`/`200`.
- [x] `.../applications/controllers/reserva/cancel-by-protocol/cancel-reserve-by-protocol.controller.ts`
      **(ALT)** — passa a validar `cancel-by-protocol.dto` no body e repassar `motivo` (o usecase
      loga/usa nas `vars` do WhatsApp).
- [x] `.../applications/routes/reserva.route.ts` — `+ reservaRoute.patch("/protocol/:number/reagendar",
      rescheduleReserveByProtocol);` (público, antes de `/:id` se necessário — o padrão atual já
      separa `/protocol/*`).

### Fase 7 — Fluxos públicos de **`ranking_agendamento`**

- [x] `.../domain/models/ranking-agendamento-auditoria.model.ts` — `+ slot_anterior?`, `slot_novo?`
      (`ISlotSnapshot` — reusar o tipo do `reserva-reagendamento-auditoria.model.ts` ou um shared).
- [x] `.../domain/ports/output/ranking-agendamento-persistence.port.ts` — `+ IRescheduleRankingAgendamentoPort`
      (`execute(id, data: { court; unit; data; hora_inicio; hora_fim }): Promise<IRankingAgendamento | null>`).
- [x] `.../domain/ports/output/ranking-agendamento-auditoria-persistence.port.ts` — `ICreate*`
      aceita `slot_anterior?`/`slot_novo?`.
- [x] `.../domain/ports/input/ranking-agendamento.input-port.ts` — `+ IFindRankingAgendamentoByProtocolUseCase`
      (retorna `IRankingAgendamento & { can_cancel; cancellation_deadline?; can_reschedule;
      reschedule_deadline? }`), `+ ICancelRankingAgendamentoByProtocolData` (`{ numero_protocolo,
      motivo }`) + `UseCase`, `+ IRescheduleRankingAgendamentoByProtocolData` (`{ numero_protocolo,
      motivo, unit, court, date, start_time, end_time }`) + `UseCase`.
- [x] `.../infra/schemas/ranking-agendamento-auditoria.schema.ts` — campos `slot_anterior`/`slot_novo`
      (`Mixed` ou subdoc, `required: false`).
- [x] `.../infra/adapters/ranking_agendamento/reschedule/reschedule-ranking-agendamento.adapter.ts`
      **(NEW)** — `findByIdAndUpdate(id, { court, unit, data, hora_inicio, hora_fim }, { new: true })`.
- [x] `.../infra/adapters/ranking_agendamento_auditoria/create/create-ranking-agendamento-auditoria.adapter.ts`
      **(ALT)** — grava `slot_anterior`/`slot_novo` quando presentes.
- [x] `.../domain/usecases/ranking-agendamento/find-by-protocol/find-ranking-agendamento-by-protocol.usecase.ts`
      **(NEW)** — `IFindRankingAgendamentoByProtocolPort` (existe) + `IFindSchedulingsByCourtUnitFromDatePort`
      (para achar o `scheduling` atual e calcular a janela) **ou** calcula direto de `data`+`hora_inicio`
      via `getSlotStartDate` (mais simples, sem I/O extra). `404` se não achar ou `status === 'pending'`.
      Devolve `can_cancel`/`can_reschedule` + deadlines (`cancelled`/`rejected`/`expired` → `false`).
- [x] `.../domain/usecases/ranking-agendamento/cancel-by-protocol/cancel-ranking-agendamento-by-protocol.usecase.ts`
      **(NEW)** — construtor: `IFindRankingAgendamentoByProtocolPort`, `ISetRankingAgendamentoStatusPort`,
      `EventSchedulingImpactService`, `ICreateRankingAgendamentoAuditoriaPort` *(opcional — registrar
      cancelamento? decisão: **não** grava auditoria de cancelamento, alinhado à reserva)*,
      `ISendWhatsappNotificationPort`. Fluxo: `404` (não achou / `pending`); `409` (já `cancelled`/
      `rejected`/`expired`); janela de 2 h (`getSlotStartDate(data, hora_inicio)`); `status:
      'cancelled'` + `releaseEventFromSchedulings`; se `status` anterior era `approved` **e**
      `cliente?.telefone` → WhatsApp `cancelamento-reembolso`.
- [x] `.../domain/usecases/ranking-agendamento/reschedule-by-protocol/reschedule-ranking-agendamento-by-protocol.usecase.ts`
      **(NEW)** — construtor: `IFindRankingAgendamentoByProtocolPort`, `IFindSchedulingsByExactSlotsPort`,
      `EventConflictService`, `IFindSchedulingsByCourtUnitFromDatePort`,
      `IFindActiveReservesForSchedulingsPort`, `EventSchedulingImpactService`,
      `IRescheduleRankingAgendamentoPort`, `ICreateRankingAgendamentoAuditoriaPort`,
      `ISendWhatsappNotificationPort`. Fluxo: `404`/`409` (status); janela 2 h (slot atual); resolve
      **1** slot novo via `findSchedulingsByExactSlots` (`409` se não existir/indisponível); valida
      conflito das 5 fontes (`eventConflictService.getConflicts` na janela do slot novo, `excludeAgendamentoId`
      = próprio) + reserva ativa; janela de 2 h no slot novo; `releaseEventFromSchedulings` (janela
      antiga) → `reschedulePort` → `applyEventToSchedulings` (janela nova); auditoria
      (`slot_anterior`/`slot_novo`, `usuario_nome = cliente?.nome ?? 'Cliente (protocolo)'`,
      `dia_anterior`/`dia_novo`); WhatsApp `reagendamento-confirmado` se `cliente?.telefone`.
- [x] `.../domain/usecases/ranking-agendamento/change-day/change-day-ranking-agendamento.usecase.ts`
      — (já alterado na Fase 5 para a guarda de 2 h).
- [x] `.../applications/dto/ranking-reschedule-by-protocol.dto.ts` **(NEW)** — `numero_protocolo`
      (path), `motivo` (body), `unit`/`court` (ObjectId), `date`, `start_time`, `end_time`.
- [x] `.../applications/dto/ranking-cancel-by-protocol.dto.ts` **(NEW)** — `numero_protocolo` (path)
      + `motivo` (body).
- [x] `.../applications/controllers/ranking_agendamento/{find-by-protocol,cancel-by-protocol,reschedule-by-protocol}/*.controller.ts`
      **(NEW)** — públicos; sem `req.databaseUser`.
- [x] `.../applications/routes/ranking-agendamento.route.ts` —
      `+ rankingAgendamentoRoute.get("/protocolo/:numero_protocolo", findRankingAgendamentoByProtocol);`
      `+ rankingAgendamentoRoute.patch("/protocolo/:numero_protocolo/cancelar", cancelRankingAgendamentoByProtocol);`
      `+ rankingAgendamentoRoute.patch("/protocolo/:numero_protocolo/reagendar", rescheduleRankingAgendamentoByProtocol);`
      **antes** de `/:id` (o arquivo já registra `/protocolo/.../comprovante` antes de `/:id` — seguir a mesma ordem).

### Fase 8 — Wiring / config (`agendamentos`)

- [x] `.../config/env.ts` / `env.spec.ts` — `WHATSAPP_*` (Fase 4).
- [x] `.../config/container.ts` — instanciar: `sendWhatsappNotificationAdapter`,
      `setReserveSchedulingsAdapter`, `createReservaReagendamentoAuditoriaAdapter` +
      `listReservaReagendamentoAuditoriaAdapter`, `rescheduleRankingAgendamentoAdapter`; novos
      usecases (`rescheduleReserveByProtocol`, `findRankingAgendamentoByProtocol`,
      `cancelRankingAgendamentoByProtocol`, `rescheduleRankingAgendamentoByProtocol`);
      `DeleteReserveUsecase` recebe `sendWhatsappNotificationAdapter`; `FindReserveByProtocolUsecase`
      sem mudança de assinatura (só a lógica interna).
- [x] `.../config/container.spec.ts` — chaves/contagem.

### Fase 9 — Infra (`beach-center-server`) + verificação local

- [x] `beach-center-server/docker-compose.dev.yml` — `+ WHATSAPP_API_URL: http://whatsapp:5003/api/v1`
      e `+ WHATSAPP_INTERNAL_API_KEY: dev-whatsapp-key` no `environment` do serviço `agendamentos`
      (conferir o nome/porta reais do serviço whatsapp no compose).
- [x] `beach-center-server/.env.dev.example` — as 2 vars (comentadas) + nota.
- [x] `beach-center-server/README.md` — tabela de env do `agendamentos`, se existir.
- [x] `agendamentos`: `npx tsc --noEmit` limpo; `npx eslint src/` limpo.
- [x] `campeonatos`: `npx tsc --noEmit` limpo; `npx eslint src/` limpo.
- [x] `beach-center-whatsapp`: `node -c` nos arquivos novos/alterados (sem suíte — decisão do
      usuário); smoke manual do `send-by-key` fica para o `/speckit-test`.
- [x] Suítes de `agendamentos` e `campeonatos` executadas — anotar specs que quebram por contrato
      (esperado: `delete-reserve`, `find-by-protocol` de reserva, `change-day` campeonato/ranking,
      `create-bulk` ranking, `find-confirmed-ranking-...`, `container`/`env` em `agendamentos`;
      `agendar-ranking`, `ranking-agendar.dto`, client `create-bulk` em `campeonatos`). Ajuste é do
      `/speckit-unit-tests`.

## Desvios / notas de implementação (`/speckit-implement`, 2026-09-10)

1. **`campeonatos` — `create-bulk` client adapter sem código novo.** O `payload` já é repassado
   inteiro no `axios.post`; adicionar `cliente?` ao tipo do payload basta. Nenhuma mudança nos
   `trocar-dia-*-client.adapter` (o `handleHttpError` genérico já propaga o `409` da janela de 2 h).
2. **RA reschedule — conflito de reserva ativa por `available`.** O reagendamento de `ranking_agendamento`
   resolve um `scheduling` **existente** (`findSchedulingsByExactSlots`) e exige `available === true`;
   isso já cobre "reserva ativa" e "dia fechado" (ambos zeram `available`), então o usecase não
   repete a varredura de reservas ativas do `change-day` — só `EventConflictService.getConflicts`
   para as exceções recorrentes.
3. **`ISlotSnapshot.scheduling_id` virou opcional.** A reserva comum sempre preenche; o
   `ranking_agendamento` (bloqueio de quadra, não amarrado a um `scheduling`) deixa ausente no
   `slot_anterior`. Tipo movido para `domain/models/slot-snapshot.model.ts` (compartilhado pelas
   duas auditorias).
4. **`reserva_reagendamento_auditorias` — só `create`.** O `list` adapter/port foi removido (sem
   endpoint consumidor nesta task); o schema/coleção existe para um endpoint admin futuro.
5. **`change-day` (campeonato + ranking) — guarda de 2 h com `ConflictError` (409).** Reusa
   `ConflictError` de `domain/errors.ts` (não uma classe nova). Propaga sem tocar o `catch` (o
   `handleUsecaseError` repassa `DomainError`).
6. **`DeleteReserveUsecase.execute` ganhou `options.motivo`.** `executeByProtocol(number, motivo?)`
   repassa; o `close-date` e o `ConflictResolutionService` seguem chamando só com
   `skipCancellationTimeValidation`. WhatsApp `cancelamento-reembolso` dispara em qualquer
   cancelamento de reserva `approved` (inclui o `close-date` com `cancel_reserves: true`) —
   coerente com D3.
7. **`beach-center-whatsapp` sem ESLint config** (serviço isento). Só `node -c` nos 5 arquivos
   novos/alterados — todos OK.
8. **Compose**: serviço `whatsapp` roda em `PORT 5004` (não 5003) e já tinha
   `WHATSAPP_INTERNAL_API_KEY: dev-whatsapp-key`. `WHATSAPP_API_URL: http://whatsapp:5004/api/v1`
   adicionado no serviço `agendamentos`. `beach-center-server/README.md` — sem mudança (não lista
   env do `agendamentos`).
9. **Specs quebrados deixados para `/speckit-unit-tests`** (6 suítes / 13 testes em `agendamentos`,
   todos por mudança de contrato): `delete-reserve.usecase` (novo port no construtor),
   `find-by-protocol.usecase` (`+ can_reschedule`, `404` em `pending`),
   `cancel-reserve-by-protocol.controller` (DTO de body), `routes.spec` (4 rotas novas),
   `container.spec` (guard "sem chaves extras" — 4 chaves novas),
   `find-confirmed-ranking-...adapter` (`$nin`). `env.spec` já ajustado. **`campeonatos`: 83/83
   suítes verdes** (o `contato` é opcional e passthrough).

## Testes unitários (`/speckit-unit-tests`, 2026-09-10)

Specs afetados corrigidos + specs novos para os arquivos novos + cobertura dos `AC-*`.
**`beach-center-whatsapp` sem testes** (decisão do usuário, `context.md` §84–85).

**`agendamentos`** — 292 suítes / 1396 testes verdes · cobertura **98.72 % stmt / 90.76 % branch**.
- Ajustados: `find-by-protocol.usecase.spec` (`404` em pending, `can_reschedule`),
  `delete-reserve.usecase.spec` (`+ ISendWhatsappNotificationPort` + AC-5/6),
  `cancel-reserve-by-protocol.controller.spec` (DTO de body + `motivo`),
  `routes.spec` (4 rotas novas, total 70→74), `container.spec` (+4 chaves),
  `find-confirmed-ranking-...adapter.spec` (`$nin` — AC-30), `env.spec` (WHATSAPP),
  `change-day` campeonato/ranking (AC-25/26), `create-bulk-ranking.usecase.spec` (AC-28/29).
- Novos: `agendamento-time-window.spec`, `reschedule-reserve-by-protocol.usecase.spec` (AC-13..21),
  `{find,cancel,reschedule}-ranking-agendamento-by-protocol.usecase.spec` (AC-4/10/11/12/22/23/24),
  `send-whatsapp-notification.adapter.spec` (AC-7 — nunca lança), `set-reserve-schedulings.adapter.spec`,
  `reschedule-ranking-agendamento.adapter.spec`, `create-reserva-reagendamento-auditoria.adapter.spec`,
  4 specs dos controllers públicos.

**`campeonatos`** — 83 suítes / 331 testes verdes · cobertura **98.63 % stmt / 88.49 % branch**.
- `agendar-ranking.usecase.spec` — `+ contato` propagado como `cliente` (AC-28) / ausente (AC-29).

`tsc --noEmit` e `eslint` limpos nos dois repos. Nenhum commit.

## Validação (`/speckit-validate`, 2026-09-10 — autônoma, a pedido da usuária)

Revisão arquivo por arquivo dos 4 repos. Correções aplicadas:
1. **`motivo` do cancelamento público virou OPCIONAL** (reserva e RA) — o frontend atual
   (`ProtocolSection`) cancela **sem corpo**; exigir `motivo` (Q1 default) quebraria o botão de
   cancelar já em produção (a modal com campo de motivo é da task de frontend futura,
   `context.md` §223–226). O `motivo` continua sendo capturado e repassado ao WhatsApp/log quando
   enviado. **Reagendamento mantém `motivo` obrigatório** (endpoint novo, sem consumidor, gravado
   na auditoria). Ajustados: `cancel-by-protocol.dto`, `ranking-cancel-by-protocol.dto`,
   `ICancelRankingAgendamentoByProtocolData`, `cancel-ranking-agendamento-by-protocol.usecase`.
2. **WhatsApp defensivo no domínio** — os 4 pontos de disparo (`delete-reserve`,
   `cancel-ranking-…by-protocol`, `reschedule-reserve-…by-protocol`, `reschedule-ranking-…by-protocol`)
   passaram a envolver a chamada ao port em `try/catch` próprio. O adapter já engole erros, mas
   assim um bug futuro no adapter **nunca** deixa uma reserva meio-cancelada / meio-reagendada.
3. **`create-many-ranking-agendamentos.adapter`** — mapeia `cliente` campo a campo em vez de
   repassar o subdocumento Mongoose direto no objeto de domínio.
Specs de cancelamento ajustados às mudanças (aceitam ausência de `motivo`). `tsc`/`eslint`/suítes
verdes nos 3 repos TS.

## Testes de componente (`/speckit-component-tests`, 2026-09-10)

**Não aplicável.** A task 006b é 100 % backend (`agendamentos` + `beach-center-whatsapp` +
`beach-center-bff-campeonatos`); `beach-center-app` **não foi tocado** e o projeto não tem
Cypress/Cucumber configurado (idêntico à 006a e às tasks 001–005). A página `/protocolo/:number`,
a modal de cancelamento com 2 opções e o aviso de reembolso são da **task de frontend futura**
(`context.md` §223–226). Sem superfície de UI/E2E para cobrir — a validação fica pelos testes
unitários (acima) e pelos cenários exploratórios do `/speckit-test`.

## Critérios de Aceite (formais)

> Given / When / Then. "reserva" = reserva comum; "RA" = `ranking_agendamento`. Base para
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.

### Buscar por protocolo (público)

**AC-1 — Reserva antes de `waiting_approve` → 404.**
- **Given** uma reserva `pending` (sem `number`, ou com `number` legado)
- **When** `GET /reservas/protocol/:number`
- **Then** `404` (`"Reserva nao encontrada"`).

**AC-2 — Reserva a partir de `waiting_approve` → dados + flags.**
- **Given** uma reserva `waiting_approve`/`approved` cujo slot mais cedo começa em > 2 h
- **When** `GET /reservas/protocol/:number`
- **Then** `200` com os dados da reserva, `can_cancel: true`, `can_reschedule: true`,
  `cancellation_deadline` e `reschedule_deadline` (iguais, 2 h antes do slot mais cedo).

**AC-3 — Flags falsas para terminais / fora da janela.**
- **Given** uma reserva `cancelled` / `rejected` / `expired`, **ou** `approved` cujo slot começa
  em < 2 h
- **When** `GET /reservas/protocol/:number`
- **Then** `can_cancel: false` e `can_reschedule: false`; os `*_deadline` ainda são devolvidos
  quando há slots.

**AC-4 — RA `pending` → 404; a partir de `waiting_approve` → dados + flags.**
- **Given** um RA `pending`
- **When** `GET /ranking-agendamentos/protocolo/:numero_protocolo`
- **Then** `404`. **Given** o mesmo RA em `waiting_approve` → `200` com `can_cancel`/`can_reschedule`
  + deadlines.

### Cancelamento público por protocolo

**AC-5 — Cancelar reserva `waiting_approve` dentro da janela.**
- **Given** uma reserva `waiting_approve`, slot em > 2 h, `motivo` informado
- **When** `PATCH /reservas/protocol/:number/cancel` com `{ "motivo": "..." }`
- **Then** `200`; `status: "cancelled"`; slots liberados; **sem** WhatsApp (não era `approved`);
  `refund_status` não setado.

**AC-6 — Cancelar reserva `approved` dispara WhatsApp de reembolso (D10).**
- **Given** uma reserva `approved`, dentro da janela
- **When** cancela por protocolo
- **Then** `status: "cancelled"`, `refund_status: "manual"`, slots liberados, **e** uma chamada
  `POST {WHATSAPP_API_URL}/messages/send-by-key` com `key: "cancelamento-reembolso"`, `to:
  reserve.phone`, `vars: { protocolo, nome }`.

**AC-7 — Falha no WhatsApp não aborta o cancelamento.**
- **Given** o cenário de AC-6, mas `whatsapp` retorna `500` / está fora do ar
- **When** cancela
- **Then** a reserva **fica cancelada** normalmente (`200`); o erro do WhatsApp é só logado.

**AC-8 — Cancelar fora da janela de 2 h → 400/409.**
- **Given** uma reserva cujo slot mais cedo começa em < 2 h
- **When** cancela por protocolo
- **Then** `"Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"`; nada muda;
  sem WhatsApp.

**AC-9 — Cancelar reserva já terminal → 409.**
- **Given** reserva `cancelled` / `expired`
- **When** cancela por protocolo
- **Then** `409` (`"Reserva ja foi cancelada"` / `"Reserva expirada"` — 006a).

**AC-10 — Cancelar RA `approved` com `cliente.telefone` → cancela + WhatsApp.**
- **Given** um RA `approved` com `cliente: { telefone }`, dentro da janela, `motivo` informado
- **When** `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/cancelar`
- **Then** `status: "cancelled"`; o bloqueio da janela é liberado; chamada `send-by-key` com
  `key: "cancelamento-reembolso"`, `to: cliente.telefone`.

**AC-11 — Cancelar RA `approved` sem `cliente.telefone` → cancela, sem WhatsApp.**
- **Given** um RA `approved` sem `cliente` (legado)
- **When** cancela por protocolo
- **Then** cancela normalmente; **nenhuma** chamada ao `whatsapp`.

**AC-12 — Cancelar RA `waiting_approve` → cancela, sem WhatsApp.**
- **Given** RA `waiting_approve`
- **When** cancela por protocolo
- **Then** `cancelled`, bloqueio liberado, sem WhatsApp (só `approved` dispara).

### Reagendamento público por protocolo — reserva

**AC-13 — Reagendar reserva `waiting_approve` (caminho feliz, 1 slot).**
- **Given** reserva `waiting_approve` de 1 slot, dentro da janela; um slot novo `{ unit, court,
  date, start_time, end_time }` que **existe** como `scheduling` disponível, não passado, sem
  conflito, começando em > 2 h; `motivo` informado
- **When** `PATCH /reservas/protocol/:number/reagendar`
- **Then** `200`; `reserve.scheduling_id` aponta para o novo `scheduling`; o slot antigo volta a
  `available: true`; o novo fica `available: false`; `number` e `total` **inalterados**; um
  registro em `reserva_reagendamento_auditorias` com `slots_anteriores`/`slots_novos`/`motivo`;
  chamada `send-by-key` `key: "reagendamento-confirmado"`.

**AC-14 — Reagendar preserva o nº de slots (2/3).**
- **Given** reserva de 2 slots
- **When** reagenda com **2** slots novos válidos
- **Then** `200`; os 2 antigos liberados, os 2 novos vinculados; `total` inalterado.

**AC-15 — Nº de slots diferente → 409.**
- **Given** reserva de 2 slots
- **When** reagenda enviando 1 (ou 3) slots
- **Then** `409` (`"O reagendamento deve manter o mesmo numero de horarios"`); nada muda.

**AC-16 — Slot novo inexistente / indisponível → 409.**
- **Given** um slot novo cujo `scheduling` não existe, **ou** existe mas `available: false` (dia
  fechado / já reservado)
- **When** reagenda
- **Then** `409` (`"Horario indisponivel para reagendamento"`); os slots antigos **permanecem**
  vinculados e bloqueados.

**AC-17 — Slot novo em conflito com bloqueador (5 fontes) → 409.**
- **Given** um slot novo que coincide com `aula_bloqueio` / `mensalista_plano` /
  `campeonato_agendamento` / `ranking_agendamento` ativo, ou evento `OUTRO` `CONFIRMED`
- **When** reagenda
- **Then** `409` (`"Um ou mais horarios selecionados possuem excecao de agendamento"`); nada muda.

**AC-18 — Slot novo no passado ou em < 2 h → 400/409.**
- **Given** um slot novo cuja data já passou, **ou** que começa em menos de 2 h a partir de agora
- **When** reagenda
- **Then** erro (`"...datas que ja passaram"` / `"...ate 2 horas antes..."`); nada muda.

**AC-19 — Reagendar fora da janela dos slots atuais → 409.**
- **Given** reserva cujo slot atual mais cedo começa em < 2 h
- **When** reagenda
- **Then** `409` (fora da janela); nada muda; sem WhatsApp.

**AC-20 — Reagendar reserva terminal → 409; reserva `pending` → 404.**
- **Given** reserva `cancelled`/`rejected`/`expired` → `409`. **Given** reserva `pending` → `404`.

**AC-21 — Falha no WhatsApp não aborta o reagendamento.**
- **Given** AC-13 mas o `whatsapp` falha
- **Then** o reagendamento conclui (`200`, auditoria gravada); erro só logado.

### Reagendamento público por protocolo — RA

**AC-22 — Reagendar RA `waiting_approve` (caminho feliz).**
- **Given** RA `waiting_approve`, dentro da janela; slot novo `{ unit, court, date, start_time,
  end_time }` existente/disponível/sem conflito/futuro; `motivo`
- **When** `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/reagendar`
- **Then** `200`; o RA passa a ter `court`/`unit`/`data`/`hora_inicio`/`hora_fim` do slot novo; o
  bloqueio da janela antiga é liberado e o da nova aplicado; `numero_protocolo` inalterado;
  registro em `ranking_agendamento_auditorias` com `slot_anterior`/`slot_novo`; `send-by-key`
  `reagendamento-confirmado` **se** `cliente?.telefone`.

**AC-23 — RA: conflito no slot novo → 409.**
- **Given** slot novo com bloqueador ou reserva ativa
- **When** reagenda
- **Then** `409` (`RankingAgendamentoConflictError`); nada muda.

**AC-24 — RA: fora da janela / terminal / `pending` → erro.**
- **Given** RA cujo slot atual começa em < 2 h → `409`. `cancelled`/`rejected`/`expired` → `409`.
  `pending` → `404`.

### Janela de 2 h nas rotas internas (task 005)

**AC-25 — `PATCH /ranking-agendamentos/:id/trocar-dia` a < 2 h → 409.**
- **Given** um RA cujo `data` + `hora_inicio` está a menos de 2 h de agora
- **When** a rota interna `trocar-dia` é chamada (`x-api-key`)
- **Then** `409` (`"Só é possível trocar o dia até 2 horas antes do horário do agendamento"`);
  nada muda.

**AC-26 — `PATCH /campeonato-agendamentos/:id/trocar-dia` a < 2 h → 409.**
- **Given** um `campeonato_agendamento` a < 2 h do jogo
- **When** a rota interna `trocar-dia` é chamada
- **Then** `409` com a mesma mensagem; nada muda (nem com `cancelar_conflitos: true`).

**AC-27 — `trocar-dia` interno a > 2 h continua funcionando.**
- **Given** agendamento a > 2 h do jogo, `motivo` informado
- **When** `trocar-dia` interno
- **Then** comportamento da task 005 preservado (troca a `data`, auditoria, conflito 409, etc.).

### `ranking_agendamento.cliente` / `contato`

**AC-28 — `POST /rankings/:id/agendar` com `contato` propaga `cliente` a todas as partidas.**
- **Given** um ranking `PARTIDAS` com N partidas pendentes; `POST /rankings/:id/agendar` com
  `contato: { nome, email, telefone }`
- **When** o `agendar-ranking` roda
- **Then** cada `ranking_agendamento` criado tem `cliente` igual ao `contato` enviado.

**AC-29 — `POST /rankings/:id/agendar` sem `contato` continua válido.**
- **Given** o mesmo, **sem** `contato`
- **When** agenda
- **Then** `201`; os `ranking_agendamento` são criados **sem** `cliente` (campo ausente).

### Kernel / regressão

**AC-30 — `ranking_agendamento` `expired` não bloqueia mais (fix 006a).**
- **Given** um RA `expired` numa quadra/data
- **When** o `EventConflictService.getConflicts` avalia um novo agendamento nessa quadra/data
- **Then** o RA `expired` **não** aparece como bloqueador (`find-confirmed-ranking-...` filtra
  `$nin: ['cancelled', 'expired']`).

**AC-31 — `tsc` e ESLint limpos.**
- **Given** `agendamentos` e `campeonatos` após a implementação
- **When** `tsc --noEmit` e `eslint`
- **Then** zero erros nos dois.

**AC-32 — `send-by-key` resolve e envia.**
- **Given** o template `reagendamento-confirmado` no banco (seed) com `{{protocolo}}` etc.
- **When** `POST /api/v1/messages/send-by-key` `{ to, key: "reagendamento-confirmado", vars: {...} }`
  com `x-api-key` correta
- **Then** `WhatsAppService.sendTextMessage` é chamado com o texto interpolado; `200`. Sem
  `x-api-key` → `401`. `key` inexistente → `404`.

**AC-33 — Seed idempotente.**
- **Given** o banco já tem `cancelamento-reembolso` com `editable_body` editado pelo admin
- **When** o serviço `beach-center-whatsapp` reinicia (roda o seed)
- **Then** o `editable_body` editado **não** é sobrescrito; templates com `key` nova são inseridos.

## Próximo passo sugerido

`/speckit-implement` — implementa o checklist (sem commit), ESLint estrito. Depois:
`/speckit-unit-tests` (specs de `agendamentos` + `campeonatos`; **`beach-center-whatsapp` sem
testes**), `/speckit-component-tests` (N/A — sem frontend), `/speckit-validate`, `/speckit-test`,
`/speckit-complete`, `/speckit-documentation`. Encerra o par 006a/006b.
