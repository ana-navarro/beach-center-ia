# Task 006a — Ciclo de vida do pagamento (reserva + partida de ranking)

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> Perguntas de regra de negócio já fechadas (3 rodadas — ver "Decisões confirmadas").
> **Parte 1 de 2.** A troca de dia / cancelamento público por protocolo é a **task 006b**.

## Título

Colocar reserva comum e `ranking_agendamento` numa **máquina de estados de pagamento estilo
e-commerce** (`pending → waiting_approve → approved`, com `expired` para não-pagos), gerar o
**protocolo só ao entrar em `waiting_approve`**, expirar reservas `pending` abandonadas e
**remover o reembolso automático** (Getnet) dos serviços.

> Frontend **fora de escopo** (task futura). O fluxo de **envio/validação de comprovante** também
> **não muda de lugar nesta task** — continua em `beach-center-bff-pagamentos`, só ajustado à nova
> máquina de estados (ver "Sistema de pagamento — futuro").

## Decisões confirmadas

| # | Decisão |
|---|---|
| **A1** | Escopo = **reserva comum E `ranking_agendamento`** (task 005). Modelados em paralelo — "quase a mesma coisa; a diferença ainda não existe, mas pode existir". |
| **D4** | **Máquina de estados (e-commerce):** `pending` (reserva criada, **sem comprovante, sem protocolo**) → `waiting_approve` (comprovante enviado; **protocolo gerado aqui**; aguardando o admin) → `approved` (admin analisou o comprovante e confirmou = "pagamento confirmado") → `rejected` / `cancelled` / **`expired`** (terminais). |
| **D5** | `IReserveStatus` **ganha `'waiting_approve'` e `'expired'`** (model + schema). `RankingAgendamentoStatus` **ganha `'expired'`** (`waiting_approve` já tem). |
| **D6** | Protocolo gerado na transição **`→ waiting_approve`**. `reserva` **nasce `pending` sem `number`**. `ranking_agendamento`: `numero_protocolo` **continua gerado no `create-bulk`** (menor diff — o anexo de comprovante é localizado por ele), mas só "conta" para o fluxo público a partir de `waiting_approve` (isso é exercido na 006b). |
| **D-exp** | Reserva `pending` **sem comprovante em 24 h** (a partir de `created_at`) → status **`expired`**, os `scheduling`(s) voltam a **disponível**. Verificação **lazy no `GET /reservas`** (sem cron nesta task). **Vale também para `ranking_agendamento` em `pending`**. |
| **B-list** | `GET /reservas`, a cada chamada: (a) roda a varredura de expiração; (b) **ordena os `waiting_approve` primeiro** (o admin precisa saber o que ainda não olhou); (c) devolve a **contagem de pendentes de aprovação**. |
| **D2** | **Reembolso automático sai.** Some **todo o caminho de reembolso**: em `agendamentos` (`payment-refund.port` + `refund-reserve-payment.adapter` + a chamada no `delete-reserve`) e em `beach-center-bff-pagamentos` (`/refunds`, `create-refund.usecase`, `getnet-refund`/`mock-refund` adapters, `create-refund.dto`, `IRefundGatewayPort`). O cancelamento passa a marcar `refund_status = 'manual'` e **cancela normalmente**. Reembolso vira **contato direto usuário↔admin** (a mensagem automática no WhatsApp é da 006b). |
| **PIX** | Daqui pra frente **só PIX**. O restante do Getnet (checkout, webhook, payment-methods) fica **congelado** em `beach-center-bff-pagamentos` até a task futura de descontinuação. |
| **PAG-mín** | O `SubmitManualPaymentUsecase` (pagamentos) passa a mover a reserva **`pending → waiting_approve`** ao persistir o comprovante (hoje não mexe no status no caminho feliz). O `ReviewManualPaymentUsecase` passa a revisar a partir de **`waiting_approve`** (hoje espera `pending`). Nada além disso muda em `pagamentos` nesta task. |

## Estado atual do código (levantado nesta task)

### `beach-center-bff-agendamentos` — reserva

- `IReserveStatus` = `'pending' | 'approved' | 'rejected' | 'cancelled'` (model). Schema Mongoose:
  `enum: [...], default: 'pending'`. **`number: { type: String, required: false }`** — o schema já
  aceita reserva sem protocolo; o **tipo** `IReserve.number: string` é que precisa virar opcional.
  `IReserveRefundStatus` = `'pending' | 'approved' | 'failed' | 'skipped'`.
- `CreateReserveUsecase`: `number: data.number || generateNumericId()` (`customAlphabet('0123456789',
  10)`), `status: 'pending'` — protocolo sempre gerado na criação hoje.
- `UpdateReserveAndSchedulingStatusUsecase` (`PATCH /reservas/:id/status`, `adminOrInternalApiKey`):
  `pending`/`approved` → bloqueia `scheduling`; qualquer outro → libera. Revalida conflito com
  exceções ao voltar para `pending`/`approved`. **É o ponto por onde o status da reserva muda**
  (chamado por `pagamentos` no review).
- `DeleteReserveUsecase.refundIfNeeded`: estorna se `status === 'approved'` && `total > 0`; `PIX`
  sem `payment_id` → `skipped`; cartão sem `payment_id` → `409`; senão `RefundReservePaymentAdapter`
  (`POST {PAGAMENTOS_API_URL}/refunds`); `failed` → `502` e **não cancela**. Janela de cancelamento
  = 2 h (`reserve-cancellation-window.ts`).
- `ListReservesUsecase`: **passthrough puro** para `listReservesPort` (sem ordenação/contagem/
  expiração).
- `scheduling_id` é **array** (até 3 — `ReserveSchedulingValidator.validateSchedulingLimit`).
- `IIsProtocolNumberTakenPort` (task 005) — consulta `reservas.number` + `ranking_agendamentos.
  numero_protocolo` (unicidade global) — **reutilizável** para gerar o protocolo tardio.
- `config/env.ts`: `PAGAMENTOS_API_URL` (obrigatória, default `localhost:5001`), `PAGAMENTOS_
  INTERNAL_API_KEY` (opcional).

### `beach-center-bff-agendamentos` — `ranking_agendamento` (task 005)

- `numero_protocolo` gerado no `create-bulk` (`status: 'pending'`), único global.
  `RankingAgendamentoStatus` = `'pending' | 'waiting_approve' | 'approved' | 'rejected' |
  'cancelled'`. `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/comprovante` (pública) →
  `409` se `status !== 'pending'` → upload → `waiting_approve`. Aprovar/rejeitar: **não existe**
  (fora da task 005). O `create-bulk` é chamado pelo `agendar-ranking` do MS de campeonatos.

### `beach-center-bff-pagamentos`

- **Comprovante:** `SubmitManualPaymentUsecase` (`POST /public/transaction-history`) — exige
  `reserve.status === 'pending'`; sobe o comprovante; cria `manual_payment` (`pending`); **no
  caminho feliz não muda o status da reserva** (só marca `rejected` em falha de upload). Preço
  esperado: `EXPECTED_AMOUNT_BY_DURATION = {1:80, 2:120, 3:160}`.
  `ReviewManualPaymentUsecase` (`PATCH /transaction-history/:id/review`) — admin aprova/rejeita;
  aprovado → `payment_method: 'PIX'` + `PATCH {agendamentosApiUrl}/reservas/:id/status`
  (`approved`); valida `reserve.status` (`pending` ou já igual ao review).
- **Reembolso:** `applications/controllers/refund/refund.controller.ts`,
  `applications/dto/create-refund.dto.ts`, `domain/usecases/refund/create-refund.usecase.ts`,
  `infra/adapters/getnet/refund/{getnet-refund,mock-refund}.adapter.ts`, rota `POST /refunds`,
  wiring no `container.ts` (`MockRefundAdapter`, `refundGateway`, `createRefund`).
  `domain/ports/output/payment-gateway.port.ts` tem `IRefundGatewayPort` + `ICreateRefundGateway
  Input` (só refund) **e** `IPaymentGatewayPort` + `ICreateCheckoutGatewayInput` (checkout — **fica**).
- **Getnet (fica congelado):** `checkout`, `payment-methods`, `webhooks/getnet`, `transaction-
  history` (o próprio comprovante).
- `agendamentos` **só chama `pagamentos` para `POST /refunds`** — nada mais. Depois desta task,
  `agendamentos` **não chama `pagamentos`** (só `pagamentos → agendamentos` no review permanece).

## Serviço(s) alvo (Princípio I)

| Serviço | Papel nesta task | Justificativa |
|---|---|---|
| **`beach-center-bff-agendamentos`** | Máquina de estados (`+ waiting_approve`, `+ expired`); protocolo gerado em `→ waiting_approve`; `waiting_approve` mantém `scheduling` bloqueado; varredura de expiração + ordenação/contagem no `GET /reservas`; remoção da porta/adapter de reembolso e da chamada no `delete-reserve` (`refund_status = 'manual'`); mesma expiração para `ranking_agendamento`. | Dono das reservas, do `ranking_agendamento` e do ponto de transição de status. |
| **`beach-center-bff-pagamentos`** | `SubmitManualPaymentUsecase` → move a reserva para `waiting_approve` ao persistir o comprovante; `ReviewManualPaymentUsecase` → revisa a partir de `waiting_approve`; **remover só o módulo de reembolso** (arquivos acima + `IRefundGatewayPort`/`ICreateRefundGatewayInput`, mantendo o checkout). README com nota de que o serviço é legado e será substituído (ver "futuro"). | É quem recebe o comprovante e move o status da reserva. |
| ~~`beach-center-app`~~ | **NÃO tocado.** | Task de frontend futura. |
| ~~`beach-center-bff-campeonatos`~~ | **NÃO tocado nesta task** — a expiração de `ranking_agendamento` roda em `agendamentos`; o `create-bulk` não muda de contrato aqui. | — |

## Regras de negócio conhecidas

1. `pending` = reserva criada, sem comprovante, **sem protocolo**.
2. Enviar comprovante (fluxo atual em `pagamentos`) → `pending → waiting_approve` **e gera o
   protocolo** (`generateNumericId` + unicidade global).
3. `waiting_approve` = aguardando o admin; `scheduling` **bloqueado** (como `pending`/`approved`).
4. Admin aprova → `approved` ("pagamento confirmado"). Admin rejeita → `rejected` (libera
   `scheduling`).
5. `pending` sem comprovante há mais de 24 h → `expired` + libera `scheduling`. Checado no
   `GET /reservas` (lazy). Idem para `ranking_agendamento` em `pending`.
6. `GET /reservas` devolve os `waiting_approve` primeiro + a contagem de pendentes de aprovação.
7. Cancelar/rejeitar/expirar **não estorna** — `refund_status` fica `'manual'`; o valor é tratado
   por contato direto usuário↔admin.

## Impacto arquitetural previsto (Arquitetura Hexagonal, Princípio II)

### `beach-center-bff-agendamentos/src/`

- `domain/models/reserva.model.ts` **(ALT)** — `IReserveStatus + 'waiting_approve' | 'expired'`;
  `number?: string` (opcional); `IReserveRefundStatus + 'manual'`.
- `domain/models/ranking-agendamento.model.ts` **(ALT)** — `RankingAgendamentoStatus + 'expired'`.
- `infra/schemas/reserva.schema.ts` / `ranking-agendamento.schema.ts` **(ALT)** — enums; garantir
  `timestamps`/`created_at`.
- `domain/usecases/reserva/create/create-reserve.usecase.ts` **(ALT)** — não gera `number`; nasce
  `pending`.
- `domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts` **(ALT)** — aceita
  `waiting_approve`; na transição `→ waiting_approve` **gera o protocolo** se ausente
  (`IIsProtocolNumberTakenPort`); `waiting_approve` conta como "bloqueia `scheduling`"
  (`shouldReleaseSchedulings` / `blockSchedulings`). `expired` = libera.
- `domain/usecases/reserva/shared/expire-pending-reserves.usecase.ts` **(NOVO)** — busca `pending`
  com `created_at` < (agora − 24 h); marca `expired`; libera `scheduling` (`SchedulingAvailability
  Service`). Reutilizável.
- `domain/usecases/reserva/list/list-reserves.usecase.ts` **(ALT)** — roda a expiração antes de
  listar; ordena `waiting_approve` primeiro; retorna `{ data, pending_approval_count }`
  (ou header). *(forma exata da resposta a fechar no plano.)*
- `domain/ports/output/*` — porta de "listar `pending` antigos" / "set status em lote"; talvez
  ajuste no `IUpdateReserveAndSchedulingStatusPort`.
- `infra/adapters/reserva/**` **(NOVO/ALT)** — adapters da expiração.
- **REMOVER** `domain/ports/output/payment-refund.port.ts`,
  `infra/adapters/payment/refund-reserve-payment.adapter.ts` (+ specs) e o wiring no
  `config/container.ts`.
- `domain/usecases/reserva/delete/delete-reserve.usecase.ts` **(ALT)** — remove `refundIfNeeded`
  (e `IRefundPaymentPort` do construtor); `refund_status = 'manual'` quando havia pagamento
  `approved`; **mantém** a liberação de `scheduling` e a janela de 2 h. *(A mensagem de WhatsApp é
  da 006b.)*
- `config/env.ts` **(ALT)** — remover `PAGAMENTOS_API_URL` / `PAGAMENTOS_INTERNAL_API_KEY` se
  ninguém mais usar (conferir `env.spec.ts`).
- Specs: nova máquina de estados; protocolo tardio + unicidade; `waiting_approve` bloqueia;
  expiração; ordenação/contagem da listagem; `delete-reserve` sem refund.

### `beach-center-bff-pagamentos/src/`

- `domain/usecases/manual-payment/submit-manual-payment.usecase.ts` **(ALT)** — no caminho feliz,
  após persistir o `manual_payment`: `updateReserveStatusPort.execute(reserve, 'waiting_approve')`.
- `domain/usecases/manual-payment/review-manual-payment.usecase.ts` **(ALT)** — aceitar
  `reserve.status === 'waiting_approve'` (além de `pending` legado) ao revisar.
- `domain/models/reserva.model.ts` (cópia local do tipo) **(ALT)** — `+ 'waiting_approve' | 'expired'`.
- **REMOVER**: `applications/controllers/refund/**`, `applications/dto/create-refund.dto.ts`,
  `domain/usecases/refund/**`, `infra/adapters/getnet/refund/**`, a rota `POST /refunds` +
  `RefundController` em `routes.ts`, o wiring de refund em `container.ts`, e
  `IRefundGatewayPort` / `ICreateRefundGatewayInput` de `payment-gateway.port.ts` (**mantendo**
  `IPaymentGatewayPort` / checkout). Conferir `IRefundResult` em `payment.model.ts` (remover se
  órfão).
- `README.md` **(ALT)** — nota: serviço legado; só PIX; comprovante e Getnet serão substituídos
  (ver "Sistema de pagamento — futuro").
- Specs: `submit` → `waiting_approve`; `review` a partir de `waiting_approve`; suíte de refund
  removida sem quebrar o resto.

## Perguntas em aberto

Nenhuma bloqueante. Defaults assumidos (ajustáveis no `/speckit-plan`):

1. Estado terminal da expiração = **`expired`** (novo valor no enum).
2. Resposta do `GET /reservas` com a contagem = **`{ data: IReserve[], pending_approval_count: number }`**
   (em vez de header).
3. `waiting_approve` e `pending` **bloqueiam** `scheduling`; `expired` **libera** (igual a
   `cancelled`/`rejected`).
4. A varredura de expiração de `ranking_agendamento` roda **onde ele for listado** (endpoint
   interno / no próprio `agendamentos`) — detalhar no plano; se não houver listagem própria, um
   usecase compartilhado chamado no mesmo ponto.
5. `pending` continua existindo como estado real (reserva começada e não paga) — **sim**.

## Sistema de pagamento — futuro (registrado aqui, NÃO faz parte desta task)

O usuário descreveu o rumo do futuro **"Sistema de pagamento"** (task própria, depois):

- **Atualizar a Constituição** com dois repos novos: `beach-center-bff-injection` e
  `beach-center-bff-llm-engine` (já existem como diretórios vazios em `services/`).
- **Tirar de `beach-center-bff-pagamentos` a inserção de comprovante** e colocar em
  **`beach-center-bff-injection`**.
- **`beach-center-bff-llm-engine`** = tudo relacionado a IA (microsserviço próprio). Usar
  **`llama3.2-vision:11b`** (Ollama, open-source/free) para validar se a imagem é mesmo um
  comprovante de pagamento.
  - Se for documento válido → armazena no Google Drive; senão → avisa o usuário pedindo um
    documento válido. **Sem comprovante válido, o agendamento nem é armazenado** — só depois do
    pagamento + comprovante válido.
  - O comprovante tem que ser **recente (data de hoje)**, não antigo, e conter os **dados corretos
    do pagamento**.
- **Aprovar/rejeitar** agendamento (fluxo com IA + humano, migrando para só-IA com o tempo):
  - **RAG** — a IA aprende com os comprovantes já aprovados para, no futuro, dispensar a aprovação
    humana.
  - Ao **aprovar** → move o comprovante para uma **pasta "aprovado"** no Drive; ao **rejeitar** →
    pasta **"rejeitado"** (base de treino futura).
- O comprovante precisa de **um identificador dentro da pasta do Drive** (o front futuro vai
  exibi-lo).
- Front futuro (só anotado — **não fazer agora**): listagem de reservas com status "concluído"
  (aprovado/recusado) / "em análise"; colunas de data/hora/local, quem realizou e quando; coluna
  de ação com um "olho" (dropdown com todas as infos + baixar comprovante) e botões **Aprovar** /
  **Recusar**. Aprovar → snackbar + aciona o service de mensageria (mensagem "aprovado" ao
  consumidor). Recusar → snackbar + alerta com **link do WhatsApp** para falar com o consumidor.

> Para **esta** task (006a): o comprovante **continua onde está** (`pagamentos`), só ajustado à
> máquina de estados. A migração para `injection`/`llm-engine` e a validação por IA são a task
> futura.

## Próximo passo

`/speckit-plan` (esta task) — depois `/speckit-implement`, testes, validate, test, complete,
documentation. A **task 006b** (troca de dia + cancelamento público por protocolo + janela de 2 h
+ WhatsApp) roda em seguida.
