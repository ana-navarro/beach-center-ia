# Task 006b — Troca de dia e cancelamento público por protocolo

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> Perguntas de regra de negócio já fechadas (3 rodadas — ver "Decisões confirmadas").
> **Parte 2 de 2.** Depende da **task 006a** (máquina de estados `pending → waiting_approve →
> approved`, protocolo tardio, `expired`, fim do reembolso).

## Título

Fluxo **público por número de protocolo** para **trocar o dia** e **cancelar** um agendamento —
para **reserva comum** e **partida de ranking** — com janela de **2 h antes do jogo**, auditoria
de reagendamento, e **aviso ao cliente por WhatsApp** (reembolso por conversa / confirmação de
troca).

> Frontend **fora de escopo** (task futura — página `/protocolo/:number`, modal de cancelamento
> com 2 opções, aviso de reembolso). Esta task entrega só a API.

## Decisões confirmadas

| # | Decisão |
|---|---|
| **A1** | Escopo = **reserva comum E `ranking_agendamento`**. |
| **B-btn** | As ações públicas (cancelar / trocar dia) ficam disponíveis a partir de **`waiting_approve`** — não dependem de `approved`. Antes de `waiting_approve` a busca por protocolo devolve **404** (não há protocolo). |
| **D8** | **Janela = 2 h antes do jogo** para **cancelar** e para **trocar o dia** (reaproveita a regra de 2 h já existente no cancelamento — `reserve-cancellation-window.ts`). |
| **D9** | As rotas **internas** de `trocar-dia` da task 005 (`PATCH /campeonato-agendamentos/:id/trocar-dia`, `PATCH /ranking-agendamentos/:id/trocar-dia`) **passam a rejeitar** troca a menos de **2 h** do `hora_inicio` do agendamento. |
| **D-resched** | **Troca de dia = refazer a seleção inteira** (unidade → quadra → data → horário). Não é "só a data". Nº de slots novos **exatamente igual** ao original (1/2/3, máx 3). Cada slot novo é validado contra as **5 fontes** (aula / mensalista / campeonato / ranking) + **dia fechado** + **disponibilidade**. Preserva **protocolo** e **identidade da reserva/pagamento**. |
| **D-resched2** | Ao confirmar: **libera** os `scheduling`(s) antigos, **cria** os novos, **re-vincula** ao registro. O `total`/preço **não muda** (nº de slots preservado; tabela por duração `{1:80, 2:120, 3:160}`). |
| **D-audit** | A troca de dia **grava auditoria**: criar coleção **`reserva_reagendamento_auditorias`** (para `ranking_agendamento` já existe `ranking_agendamento_auditorias`). Campos mínimos: registro, dia/slots anteriores, dia/slots novos, `created_at` (+ `motivo`? — ver perguntas). |
| **D3** | Ao **cancelar** um agendamento **com pagamento `approved`** → dispara mensagem no **WhatsApp** para o telefone do cliente avisando que o reembolso será tratado por conversa direta. |
| **D3b** | Ao **trocar o dia** com sucesso → dispara mensagem no **WhatsApp** de **confirmação** ("quanto mais mostrar que a ação do usuário foi validada, melhor"). |
| **D-msg** | As mensagens vivem no **banco do `beach-center-whatsapp`** (coleção `whatsapp_message_templates`, **que já existe**: `{ key, description, title, body, editable_body, fixed_body, editable, variables, active }`). Esta task faz o **seed** dos templates e o **envio por `key`**. O bot open-source, a edição no front e a config de sender em banco = **task futura** — por ora o número sender fica via env (como já é) e um número de teste. |
| **H** | `ranking_agendamento` passa a guardar **`cliente: { nome, email, telefone }`**, **enviados pelo MS de campeonatos no `create-bulk`** (não capturados no anexo do comprovante). Sem esses dados não dá para (a) mostrar na busca por protocolo, (b) disparar WhatsApp no cancelamento de ranking. |
| **D10** | Revisar/testar o fluxo de **cancelamento**: cancela corretamente **e** dispara a mensagem de reembolso-por-conversa. |

## Estado atual do código (levantado nesta task)

### `beach-center-bff-agendamentos` — reserva

- Públicas por protocolo (`reserva.route.ts`, **sem** `authMiddleware`):
  `GET /reservas/protocol/:number` → `FindReserveByProtocolUsecase` (devolve `can_cancel` +
  `cancellation_deadline`); `PATCH /reservas/protocol/:number/cancel` →
  `DeleteReserveUsecase.executeByProtocol`.
- Janela de cancelamento: **2 h** (`domain/usecases/shared/reserve-cancellation-window.ts`,
  `getCancellationDeadline` sobre o `start_time` do `scheduling` mais próximo;
  `isWithinCancellationWindow`).
- `scheduling_id` é **array** (até 3).
- Kernel `EventConflictService` (5 fontes: `EVENT`, `MENSALISTA`, `AULA`, `CAMPEONATO`,
  `RANKING`) + `close-date` — validação de conflito de uma reserva nova já usa exatamente isso
  (`ReserveSchedulingValidator.validateSchedulingsDoNotConflictWithExceptions` /
  `validateSchedulingsAreAvailable`). **Reaproveitar** na troca de dia.
- **Não existe** troca de dia / reagendamento para reserva comum.
- `DeleteReserveUsecase` — **após a 006a** já não chama reembolso; `refund_status = 'manual'`.

### `beach-center-bff-agendamentos` — `ranking_agendamento` / `campeonato_agendamento` (task 005)

- `ranking_agendamento`: `PATCH /ranking-agendamentos/:id/trocar-dia` (**interna**, `x-api-key`) —
  troca só a `data`, auditoria em `ranking_agendamento_auditorias`, conflito simples `409`,
  **sem janela de tempo**, `usuario_nome` no corpo. `PATCH /ranking-agendamentos/:id/delete`
  (**interna**). **Não há** `GET` público por protocolo nem cancelamento público.
  Modelo **não guarda** nome/e-mail/telefone (→ H).
- `campeonato_agendamento`: `PATCH /campeonato-agendamentos/:id/trocar-dia` (**interna**) — fluxo
  `409` + `cancelar_conflitos`, auditoria, **sem janela de tempo**.
- Auditoria de troca de dia: modelos `campeonato-agendamento-auditoria.model.ts` /
  `ranking-agendamento-auditoria.model.ts` + coleções `*_auditorias`.

### `beach-center-bff-campeonatos` (task 005)

- `agendar-ranking.usecase.ts` chama o `create-bulk` de `ranking_agendamento` com
  `{ id_ranking, quadras, datas, hora_inicio, duracao_partida_minutos, modalidade, quantidade }`.
  **Não passa dados de cliente.** As partidas têm `lado_a`/`lado_b` (participantes + telefones).
- `infra/adapters/{campeonato,ranking}-agendamento-client/{create-bulk,delete-bulk,trocar-dia}/
  *.adapter.ts` — `axios` + `x-api-key`; `trocar-dia` já trata `409` (conflito). Precisará tratar
  o novo `4xx` de "fora da janela de 2 h" (provável: só propagar).

### `beach-center-whatsapp` — **já tem sistema de templates**

- Coleção `whatsapp_message_templates` (`message-template.schema.js`): `{ key (único), description,
  title, body, editable_body, fixed_body, editable, variables, active }`.
  `MessageTemplateService.composeBody()` monta `body` de `editable_body` + `fixed_body`.
- `GET /message-templates` (ADMIN), `PATCH /message-templates/:key` (ADMIN — edita `editable_body`).
- `POST /messages/send-text` — `{ to, text }`, `x-api-key` = `WHATSAPP_INTERNAL_API_KEY` →
  `WhatsAppService.sendTextMessage` (**Meta / WhatsApp Cloud API** `graph.facebook.com`; sender por
  env `WHATSAPP_ACCESS_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID`).
- Serviço **JS** — **sem exigência de Arquitetura Hexagonal nem testes unitários** (decisão do
  usuário).
- `agendamentos` **não chama** o `whatsapp` hoje.

## Serviço(s) alvo (Princípio I)

| Serviço | Papel nesta task | Justificativa |
|---|---|---|
| **`beach-center-bff-agendamentos`** | Fluxo público por protocolo de **troca de dia** (reserva + ranking) e **cancelamento** (novo p/ ranking); `GET` público por protocolo p/ ranking; `can_reschedule`/`reschedule_deadline` no `find-by-protocol`; janela de 2 h no público **e** nas rotas internas da task 005; auditoria de reagendamento de reserva; disparo de WhatsApp (adapter) no cancelamento e na troca; `ranking_agendamento` ganha `cliente`. | Dono das reservas, do `ranking_agendamento`, do kernel de conflito e das rotas de `trocar-dia`. |
| **`beach-center-whatsapp`** | **Seed** dos `whatsapp_message_templates` (mensagens desta task + as previstas para o bot); **envio por `key`** (`POST /messages/send-by-key` `{ to, key, vars }` — resolve o template e envia) — **ou** manter `send-text` e o `agendamentos` compõe o texto (fechar no plano). Sem hexagonal/testes. | Já tem o esqueleto de templates + `send-text`. |
| **`beach-center-bff-campeonatos`** | `agendar-ranking` passa `cliente: { nome, email, telefone }` no `create-bulk` (por partida — fonte a fechar); `trocar-dia-*-client.adapter` propaga o `4xx` de janela de 2 h. | É quem cria e reagenda partidas de ranking. |
| ~~`beach-center-app`~~ | **NÃO tocado.** | Task de frontend futura. |
| ~~`beach-center-bff-pagamentos`~~ | **NÃO tocado nesta task** (só na 006a). | — |

## Regras de negócio conhecidas

1. **Buscar por protocolo** (público): só a partir de `waiting_approve` (404 antes). Devolve dados
   do agendamento + `can_cancel`/`cancellation_deadline` + `can_reschedule`/`reschedule_deadline`.
2. **Cancelar** (público, por protocolo): cancela; libera `scheduling`; **não estorna** (006a);
   se `status === 'approved'` → dispara WhatsApp para o telefone do cliente. Só até **2 h** antes
   do `start_time` do slot mais próximo.
3. **Trocar o dia** (público, por protocolo): cliente escolhe unidade → quadra → data → horário,
   **mesmo nº de slots** do original; valida cada slot (5 fontes + dia fechado + disponibilidade +
   não-passado + fora da janela de 2 h também no slot novo); libera os slots antigos, cria os
   novos, re-vincula; **protocolo/pagamento intactos**; grava auditoria; dispara WhatsApp de
   confirmação. Só até **2 h** antes (medido nos slots **atuais**).
4. **Rotas internas de `trocar-dia`** (campeonato/ranking, task 005): `+` guarda de 2 h antes do
   `hora_inicio` do agendamento (rejeita com `4xx` claro).
5. **`ranking_agendamento`** ganha `cliente: { nome, email, telefone }` (vindo do MS de
   campeonatos). Cancelamento público de ranking dispara WhatsApp só se houver `cliente.telefone`
   (regra de `approved` a fechar — ver perguntas).

## Impacto arquitetural previsto (Arquitetura Hexagonal, Princípio II)

### `beach-center-bff-agendamentos/src/`

**Janela de 2 h reutilizável**
- `domain/usecases/shared/reserve-cancellation-window.ts` **(ALT)** — generalizar para uma janela
  de 2 h reutilizável (cancelar + reagendar; público + interno), ou novo
  `agendamento-time-window.ts` (`getTimeWindowDeadline(startTime, hours)` /
  `isWithinWindow(deadline)`), mantendo `getCancellationDeadline` como wrapper.

**Troca de dia por protocolo — reserva**
- `domain/models/reserva-reagendamento-auditoria.model.ts` **(NOVO)** + `infra/schemas/
  reserva-reagendamento-auditoria.schema.ts` **(NOVO)** + `infra/adapters/reserva_reagendamento_
  auditoria/{create,list}/*.adapter.ts` **(NOVO)** + porta de output.
- `domain/usecases/reserva/reschedule-by-protocol/reschedule-reserve-by-protocol.usecase.ts`
  **(NOVO)** — busca por protocolo; `status ≥ waiting_approve` (404/409 senão); janela de 2 h
  sobre os slots **atuais**; `novos.length === atuais.length`; para cada slot novo valida
  (`EventConflictService` + `close-date` + disponibilidade + não-passado + janela de 2 h); cria os
  `scheduling` novos; re-vincula `reserve.scheduling_id`; libera os antigos
  (`SchedulingAvailabilityService`); mantém `number`; grava auditoria; dispara WhatsApp.
- `domain/ports/input/reserva.input-port.ts` **(ALT)** — `+ IRescheduleReserveByProtocolUseCase`;
  `FindReserveByProtocolUsecase` **(ALT)** — `+ can_reschedule` / `reschedule_deadline`.
- `domain/ports/output/*` — porta de criação de `scheduling` novo (se não houver uma adequada);
  porta de auditoria.
- `applications/dto/reschedule-by-protocol.dto.ts` **(NOVO)** — `{ number, slots: [{ unit, court,
  date, start_time, end_time }] }` (1..3).
- `applications/controllers/reserva/reschedule-by-protocol/*` **(NOVO)** — público.
- `applications/routes/reserva.route.ts` **(ALT)** — `+ PATCH /reservas/protocol/:number/reagendar`
  (público). *(o `GET /protocol/:number` e o `/cancel` já existem; só o `find-by-protocol` ganha
  campos.)*

**Troca de dia + cancelamento por protocolo — ranking**
- `domain/models/ranking-agendamento.model.ts` + `infra/schemas/ranking-agendamento.schema.ts`
  **(ALT)** — `+ cliente: { nome; email; telefone }`.
- `domain/usecases/ranking-agendamento/create-bulk/*` + `dto` **(ALT)** — receber `cliente` por
  slot no payload do `create-bulk`.
- `domain/usecases/ranking-agendamento/find-by-protocol/*` **(NOVO)** — público (hoje só há o
  interno para o `attach-proof`); expõe dados + `can_cancel`/`can_reschedule` + deadlines.
- `domain/usecases/ranking-agendamento/cancel-by-protocol/*` **(NOVO)** — público; `status ≥
  waiting_approve`; janela 2 h; `status: 'cancelled'` + libera `scheduling` + auditoria? +
  WhatsApp.
- `domain/usecases/ranking-agendamento/reschedule-by-protocol/*` **(NOVO)** — público; 1 slot;
  refaz unidade/quadra/data/horário; janela 2 h; conflito; libera antigo, cria novo; grava
  `ranking_agendamento_auditorias`; WhatsApp.
- `applications/routes/ranking-agendamento.route.ts` **(ALT)** — `+ GET /ranking-agendamentos/
  protocolo/:numero_protocolo` `+ PATCH .../protocolo/:numero_protocolo/cancelar` `+
  PATCH .../protocolo/:numero_protocolo/reagendar` (públicos).
- `applications/controllers/ranking_agendamento/*` **(NOVO)**.

**Janela de 2 h nas rotas internas (task 005)**
- `domain/usecases/campeonato-agendamento/change-day/change-day-campeonato-agendamento.usecase.ts`
  **(ALT)** — `+` guarda de 2 h sobre `parseLocalDate(data)` + `hora_inicio` (rejeita com erro
  `4xx` dedicado — `409` ou novo `AgendamentoForaDaJanelaError`).
- `domain/usecases/ranking-agendamento/change-day/change-day-ranking-agendamento.usecase.ts`
  **(ALT)** — idem.

**Integração WhatsApp**
- `domain/ports/output/whatsapp-notification.port.ts` **(NOVO)** — `ISendWhatsappNotificationPort`
  (`{ to, key, vars }` ou `{ to, text }`).
- `infra/adapters/whatsapp/send-whatsapp-notification.adapter.ts` **(NOVO)** —
  `POST {WHATSAPP_API_URL}/messages/send-by-key` (ou `/send-text`), `x-api-key`. **Non-blocking**:
  falha no envio **não** aborta o cancelamento/troca (log + segue).
- `config/env.ts` **(ALT)** — `+ WHATSAPP_API_URL`, `+ WHATSAPP_INTERNAL_API_KEY`.
- `config/container.ts` **(ALT)** — wiring dos novos usecases/adapters.

### `beach-center-bff-campeonatos/src/`
- `domain/usecases/ranking/agendar/agendar-ranking.usecase.ts` + `domain/ports/output/
  ranking-agendamento-client.port.ts` + `infra/adapters/ranking-agendamento-client/create-bulk/*`
  **(ALT)** — incluir `cliente: { nome, email, telefone }` no payload do `create-bulk` (por
  partida — derivado do responsável do `lado_a`? campo novo em `agendar-ranking`? — ver perguntas).
- `infra/adapters/{campeonato,ranking}-agendamento-client/trocar-dia/*.adapter.ts` **(ALT)** —
  propagar o `4xx` de janela de 2 h (provável: nenhum código novo — o `handleHttpError` genérico
  já cobre; confirmar no plano).

### `beach-center-whatsapp/src/`
- Seed dos `whatsapp_message_templates` (script/migration): mensagens **desta task**
  (`cancelamento-reembolso`, `reagendamento-confirmado`) + as previstas para o bot futuro
  (`boas-vindas` c/ botões 1–7, `day-use`, `agendamento-quadra`, `pagamento-confirmado`) — só como
  registros iniciais.
- `applications/routes/message.route.js` + controller **(NOVO/ALT)** — `POST /messages/send-by-key`
  `{ to, key, vars }` → busca o template por `key`, interpola `vars`, chama
  `WhatsAppService.sendTextMessage`. *(Ou manter só `send-text` — decidir no plano.)*
- Número sender de teste (por ora via env; config em banco = task futura).

### Testes (`/speckit-unit-tests`)
- `reschedule-by-protocol` (reserva + ranking): `status ≥ waiting_approve`, janela 2 h, contagem
  de slots, conflito (5 fontes + dia fechado + disponibilidade), release + create de `scheduling`,
  auditoria, protocolo/`total` preservados.
- `cancel-by-protocol` de ranking; `find-by-protocol` de ranking.
- `delete-reserve` (revisão D10): cancela + `refund_status: 'manual'` + dispara WhatsApp (mock)
  quando `approved` + é non-blocking se o WhatsApp falha.
- `change-day` campeonato/ranking com a guarda de 2 h.
- `beach-center-bff-campeonatos`: `agendar-ranking` com `cliente`.
- **`beach-center-whatsapp`: sem testes** (decisão do usuário).

## Decisões (rodada final — todas fechadas)

| # | Decisão |
|---|---|
| **Q1** | **A troca de dia (e o cancelamento) público captura `motivo` do cliente** (texto). A auditoria de reagendamento grava `motivo` + antes/depois + `created_at`. Definir `motivo` como obrigatório ou opcional fica para o `/speckit-plan` (default: obrigatório, curto). |
| **Q2** | **Cancelamento público de `ranking_agendamento` dispara WhatsApp** só quando `status === 'approved'` **e** há `cliente.telefone` — mesma regra da reserva. |
| **Q3** | **Fonte do `cliente` do `ranking_agendamento` = um contato explícito no `POST /rankings/:id/agendar`** (MS de campeonatos), aplicado a todas as partidas daquele lote. Campo novo no `agendar-ranking` DTO/usecase → repassado no `create-bulk`. |
| **Q4** | **Criar `POST /messages/send-by-key`** no `beach-center-whatsapp` — recebe `{ to, key, vars }`, resolve o template (`whatsapp_message_templates`), interpola `vars` e envia. O `agendamentos` chama por `key`, nunca compõe texto. |
| **Q5** | **Não é preciso revisão dos textos.** Redigir os padrões de `cancelamento-reembolso` e `reagendamento-confirmado` (PT-BR), fazer o seed no banco como `editable` (o admin ajusta depois via `PATCH /message-templates/:key`). O texto de reembolso instrui o cliente a **responder a própria conversa** para tratar o valor com o admin (sem link externo). |

## Notas

- ⚠️ **(pedido do usuário)** O frontend atual (`ProtocolSection`) **não tem** modal de cancelamento
  com 2 opções nem aviso de reembolso — só um `Swal` sobre a regra de 2 h. A página
  `/protocolo/:number`, a modal e o aviso de reembolso são da **task de frontend futura**.
- `beach-center-app` **não tem suíte de testes** (viola o Princípio III) — task de frontend.
- **Sistema de mensageria completo do WhatsApp** (bot open-source, edição no front, config de
  sender em banco) e **e-mail de lembrete de pagamento** = **tasks futuras**.
- Depende da **006a** ter rodado antes (máquina de estados, `expired`, fim do reembolso).

## Próximo passo

Rodar a **006a** primeiro. Depois `/speckit-plan` desta (006b).
