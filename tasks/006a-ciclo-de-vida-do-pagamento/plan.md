# Plano — 006a Ciclo de vida do pagamento (reserva + partida de ranking)

> Gerado por `/speckit-plan`. **Não implementa nada.** Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.
> Todas as regras de negócio do `context.md` estão fechadas (3 rodadas). As "Perguntas em aberto"
> do `context.md` são resolvidas aqui na seção **Decisões de design desta etapa**.
> **Parte 1 de 2.** A troca de dia / cancelamento público por protocolo + janela de 2 h + WhatsApp
> é a **task 006b**, que depende desta.

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Serviço | Situação | Mudança nesta task |
|---|---|---|
| `services/beach-center-bff-agendamentos` | produção, hexagonal (tasks 001/004/005) | Máquina de estados de pagamento da **reserva comum** e do **`ranking_agendamento`**: `+ 'waiting_approve'` e `+ 'expired'` no enum da reserva (model + schema), `+ 'expired'` no enum do ranking. Protocolo da reserva passa a ser gerado **tardiamente**, na transição `→ waiting_approve` (não mais na criação). `waiting_approve` bloqueia `scheduling` (igual a `pending`/`approved`); `expired` libera (igual a `cancelled`/`rejected`). Varredura **lazy** de expiração (24 h sem comprovante) + ordenação (`waiting_approve` primeiro) + contagem de pendentes no `GET /reservas`; mesma varredura para `ranking_agendamento` `pending` no `GET /ranking-agendamentos`. **Remoção completa** do caminho de reembolso automático (porta + adapter + chamada no `delete-reserve`); cancelamento passa a marcar `refund_status = 'manual'`. |
| `services/beach-center-bff-pagamentos` | produção, hexagonal | `SubmitManualPaymentUsecase` move a reserva `pending → waiting_approve` (gerando o protocolo do lado de `agendamentos`) **antes** de subir o comprovante, e usa o `number` retornado como prefixo do arquivo. `ReviewManualPaymentUsecase` passa a revisar a partir de `waiting_approve` (além do `pending` legado). **Remoção completa** do módulo de reembolso (controller + dto + usecase + adapters Getnet/mock + rota `POST /refunds` + `IRefundGatewayPort`/`ICreateRefundGatewayInput` + tipos órfãos), **mantendo** o checkout. `README.md` ganha nota de serviço legado (só PIX; comprovante e Getnet serão substituídos — ver "Sistema de pagamento — futuro" no `context.md`). |
| ~~`beach-center-app`~~ | — | **NÃO tocado.** Task de frontend futura. |
| ~~`beach-center-bff-campeonatos`~~ | — | **NÃO tocado.** O contrato do `create-bulk` de `ranking_agendamento` não muda; a expiração roda em `agendamentos`. |

### Stack (idêntica ao restante dos dois serviços)

Node + TypeScript, Express, Mongoose/MongoDB, `yup`, `firebase-admin`, `nanoid`
(`customAlphabet('0123456789', 10)` — mesmo padrão já usado em `CreateReserveUsecase` e no
`create-bulk` de `ranking_agendamento`), Jest + ts-jest (`coverageThreshold` global 80%), ESLint
flat config com `no-restricted-imports`.

### Estado atual relevante (levantado no `context.md` e reconferido neste plano)

- **`IReserveStatus`** = `'pending' | 'approved' | 'rejected' | 'cancelled'` (`domain/models/reserva.model.ts`).
  `IReserveRefundStatus` = `'pending' | 'approved' | 'failed' | 'skipped'`.
  `IReservaDocument.status` repete a união literal no schema; enum Mongoose idem; `number:
  { required: false }` no schema, mas `IReserve.number: string` (obrigatório no tipo).
  `reserveSchema` tem `{ timestamps: true }` → `createdAt` existe no Mongo, mas `toDomainReserve`
  **não** o expõe no domínio.
- **`CreateReserveUsecase`**: `number: data.number || generateNumericId()`, `status: 'pending'` —
  protocolo sempre gerado na criação hoje. `ICreateReserveUseCaseData` já tem `number?` opcional.
- **`UpdateReserveAndSchedulingStatusUsecase`** (`PATCH /reservas/:id/status`, `adminOrInternalApiKey`)
  — único ponto de transição de status da reserva (chamado por `pagamentos` no submit/review).
  `shouldReleaseSchedulings(status)` = `status !== 'pending' && status !== 'approved'`. Revalida
  passado/conflito/disponibilidade ao ir para `pending`/`approved`.
- **`ListReservesUsecase`**: passthrough puro para `IListReservesPort` (`ReserveModel.find(params)`).
  Controller responde `{ message, data }`.
- **`DeleteReserveUsecase`**: `refundIfNeeded` estorna se `status === 'approved' && total > 0` via
  `IRefundPaymentPort` → `RefundReservePaymentAdapter` (`POST {PAGAMENTOS_API_URL}/refunds`);
  `failed` → 502 e não cancela. Janela de 2 h em `reserve-cancellation-window.ts` (fica).
- **Disponibilidade de `scheduling`**: `find-active-reserves-for-schedulings.adapter.ts` e
  `has-active-reserve-for-scheduling.adapter.ts` filtram `status in ['pending', 'approved']` —
  central para conflito/bloqueio. `has-reserve-for-scheduling.adapter.ts` (sem filtro de status)
  não muda.
- **`IIsProtocolNumberTakenPort`** (`domain/ports/output/protocol-uniqueness.port.ts`) +
  `IsProtocolNumberTakenAdapter` — consulta `reservas.number` **e** `ranking_agendamentos.numero_protocolo`
  (unicidade global). Já instanciado no container. **Reutilizável** para o protocolo tardio.
- **`ranking_agendamento`** (task 005): `RankingAgendamentoStatus` = `'pending' | 'waiting_approve'
  | 'approved' | 'rejected' | 'cancelled'` (schema idem). Criado `pending` pelo `create-bulk`
  (`POST /ranking-agendamentos`, interno), com `numero_protocolo` já gerado, **bloqueando**
  `scheduling` via `EventSchedulingImpactService.applyEventToSchedulings`. `PATCH
  /ranking-agendamentos/protocolo/:numero_protocolo/comprovante` (pública) → `409` se `status !==
  'pending'` → upload → `waiting_approve`. `GET /ranking-agendamentos` (interno) = passthrough.
- **`beach-center-bff-pagamentos`**:
  - `SubmitManualPaymentUsecase` (`POST /public/transaction-history` e autenticada) — exige
    `reserve.status === 'pending'`; valida valor (`EXPECTED_AMOUNT_BY_DURATION = {1:80,2:120,3:160}`)
    e slots; sobe comprovante (`fileStoragePort.uploadProof({ ..., prefix: reserve.number })`);
    cria `manual_payment` (`pending`, guarda `reserve_number`). **No caminho feliz não mexe no
    status da reserva** (só seta `rejected` se o upload/persist falhar).
  - `ReviewManualPaymentUsecase` (`PATCH /transaction-history/:id/review`) — admin aprova/rejeita;
    aceita `reserve.status` `'pending'` ou já igual ao review; aprovado → grava `payment_method:
    'PIX'` + `PATCH {agendamentosApiUrl}/reservas/:id/status`.
  - **Reembolso**: `applications/controllers/refund/refund.controller.ts`,
    `applications/dto/create-refund.dto.ts`, `domain/usecases/refund/create-refund.usecase.ts`,
    `infra/adapters/getnet/refund/{getnet-refund,mock-refund}.adapter.ts`, rota `POST /refunds`
    (`authOrInternalApiKey`), wiring `MockRefundAdapter`/`refundGateway`/`createRefund` no
    `container.ts`, `IRefundGatewayPort` + `ICreateRefundGatewayInput` em `payment-gateway.port.ts`,
    `ICreateRefundInput`/`ICreateRefundResult`/`ICreateRefundUseCase` + `RefundStatus` em
    `payment.input-port.ts`, `IRefundResult` + `RefundStatus` em `payment.model.ts`.
  - `agendamentos` só chama `pagamentos` para `POST /refunds`. Depois desta task, **`agendamentos`
    não chama `pagamentos`** — só resta `pagamentos → agendamentos` (`/reservas/:id/status`).
- **Env `PAGAMENTOS_API_URL` / `PAGAMENTOS_INTERNAL_API_KEY`** aparecem, além de `agendamentos`
  (`config/env.ts` + `env.spec.ts` + `container.spec.ts`), em `beach-center-server/.env.dev.example`,
  `beach-center-server/docker-compose.dev.yml`, `beach-center-server/README.md`,
  `beach-center-app/.env.example` e `beach-center-app/src/shared/services/Api/api.service.ts`
  (o front tem sua própria base de URL — **fora de escopo**, não tocar). A doc
  `beach-center-documentation/beach-center-bff-pagamentos/reembolso.md` é atualizada pelo
  `/speckit-documentation`, não aqui.

## Decisões de design desta etapa (fecham as "Perguntas em aberto" do `context.md`)

**DD1 — Estado terminal da expiração = `'expired'` (novo valor).** Adicionado a `IReserveStatus`
(model + `IReservaDocument` + enum Mongoose) **e** a `RankingAgendamentoStatus` (model + enum
Mongoose). `waiting_approve` já existe em `ranking_agendamento`; passa a existir também em
`IReserveStatus`. Widening 100% aditivo — nenhum call site existente produz esses valores hoje;
`tsc --noEmit` MUST continuar limpo (o schema literal e o enum são alargados **junto** com o
tipo de domínio, evitando o descasamento que barrou o widening na task 005).

**DD2 — Bloqueio de `scheduling` por status.**
- **Bloqueiam** (agendamento indisponível): `pending`, `waiting_approve`, `approved`.
- **Liberam** (agendamento volta a disponível): `cancelled`, `rejected`, `expired`.
- `UpdateReserveAndSchedulingStatusUsecase.shouldReleaseSchedulings` reescrito como allowlist
  explícita de liberação (`['cancelled', 'rejected', 'expired']`), não mais `!== pending && !== approved`.
- O branch de revalidação (passado/conflito/disponibilidade) passa a cobrir também
  `waiting_approve`; o conjunto "estava terminal, revalidar disponibilidade" ganha `'expired'`.
- `find-active-reserves-for-schedulings.adapter.ts` e `has-active-reserve-for-scheduling.adapter.ts`
  passam a filtrar `status in ['pending', 'waiting_approve', 'approved']`.

**DD3 — Protocolo tardio.**
- `CreateReserveUsecase` **para de gerar `number`**: persiste `number: data.number` (só se o
  cliente mandou um explícito — caso raro/legado) e `status: 'pending'`. `ICreateReserveData.number`
  e `IReserve.number` viram **opcionais** (`number?: string`). O schema já aceita (`required: false`).
- Na transição `→ waiting_approve` dentro de `UpdateReserveAndSchedulingStatusUsecase`: se
  `reserve.number` estiver ausente, gera com `customAlphabet('0123456789', 10)` num laço
  `while (await isProtocolNumberTakenPort.execute(candidato)) regenerar` (unicidade global, mesmo
  padrão do `create-bulk` de ranking) e persiste via **novo port** `ISetReserveProtocolNumberPort`
  (um adapter dedicado — "um adapter por ação"). Idempotente: se já tem `number`, não regenera.
- `ranking_agendamento`: `numero_protocolo` **continua** sendo gerado no `create-bulk` (menor
  diff — o anexo de comprovante é localizado por ele). Nada muda aqui nesta task; o "só conta a
  partir de `waiting_approve`" é exercido na 006b.

**DD4 — Expiração lazy da reserva (`D-exp` / `B-list`).**
- Janela: `pending` **sem `comprovante`** (i.e. `status` ainda `pending`) com `createdAt` <
  `agora − 24 h`. Como `pending` só sai de `pending` quando o comprovante entra, basta filtrar
  `status: 'pending'` + `createdAt`.
- **Novo port** `IListPendingReservesOlderThanPort.execute(cutoff: Date): Promise<IReserve[]>`
  (`ReserveModel.find({ status: 'pending', createdAt: { $lt: cutoff } })`) — o filtro por
  `createdAt` vive no adapter; **não** é preciso expor `createdAt` no domínio.
- **Novo usecase** `ExpirePendingReservesUsecase` (`domain/usecases/reserva/shared/`): busca os
  candidatos e, para cada um, chama o `UpdateReserveAndSchedulingStatusUsecase` já existente com
  `'expired'` (reaproveita status + liberação de `scheduling` + revalidação num único ponto).
  Retorna a contagem de expirados (para log/observabilidade; não vai na resposta HTTP).
- `ListReservesUsecase` passa a: (1) rodar `ExpirePendingReservesUsecase` antes de listar;
  (2) buscar a lista; (3) ordenar `waiting_approve` primeiro (demais mantêm a ordem do banco);
  (4) retornar `{ data: IReserve[]; pending_approval_count: number }` (contagem = itens com
  `status === 'waiting_approve'` **após** a varredura).
- `IListReservesUseCase.execute` muda o tipo de retorno para o objeto acima; o controller
  responde `{ message, data, pending_approval_count }` (a chave `data` continua sendo o array —
  retrocompatível para quem só lê `data`).

**DD5 — Expiração lazy do `ranking_agendamento`.**
- Simétrica à DD4. **Novo port** `IListPendingRankingAgendamentosOlderThanPort.execute(cutoff:
  Date)` (`RankingAgendamentoModel.find({ status: 'pending', createdAt: { $lt: cutoff } })` — o
  schema tem `{ timestamps: true }`).
- **Novo usecase** `ExpirePendingRankingAgendamentosUsecase`
  (`domain/usecases/ranking-agendamento/shared/`): para cada candidato, `setRankingAgendamentoStatusPort`
  → `'expired'` **e** `EventSchedulingImpactService.releaseEventFromSchedulings` (janela exata do
  ranking — `specific_date = data`, `hora_inicio`–`hora_fim`, `court`, `unit`), reaproveitando o
  mesmo serviço usado no `delete-bulk` de ranking.
- Ponto de execução: `ListRankingAgendamentosUsecase` roda a varredura antes de listar (mesmo
  racional do `GET /reservas` — é o único endpoint de listagem do recurso, chamado pelo MS de
  campeonatos). **Sem cron** nesta task. A resposta do `GET /ranking-agendamentos` **não** ganha
  contagem (só a reserva tem o fluxo de admin que precisa disso).

**DD6 — Remoção do reembolso automático (`D2`).**
- `agendamentos`: `DeleteReserveUsecase` perde o parâmetro `refundPaymentPort` do construtor e o
  método `refundIfNeeded`. Quando havia pagamento (`status === 'approved' && total > 0`), passa a
  chamar `deleteReservePort.execute(id, { refund_status: 'manual' })`. **Mantém** intactas a
  liberação de `scheduling`, a janela de 2 h e `executeByProtocol`.
- `IReserveRefundStatus` ganha `'manual'` (model + enum Mongoose).
- **Apagados** de `agendamentos`: `domain/ports/output/payment-refund.port.ts`,
  `infra/adapters/payment/refund-reserve-payment.adapter.ts` (+ `.spec.ts`); wiring no
  `config/container.ts` (import + `refundReservePaymentAdapter` + arg do `DeleteReserveUsecase`);
  `IRefundPaymentPort` do `delete-reserve.usecase.ts`. `IRefundOutcome` **fica** (ainda é o shape
  de `refundData` aceito pelo `IDeleteReservePort`).
- **Apagados** de `pagamentos`: `applications/controllers/refund/**`,
  `applications/dto/create-refund.dto.ts`, `domain/usecases/refund/**`,
  `infra/adapters/getnet/refund/**`; a rota `POST /refunds` + `RefundController` +
  `refundController` em `routes.ts`; `MockRefundAdapter`/`refundGateway`/`createRefund` +
  `IRefundGatewayPort` em `container.ts`; `IRefundGatewayPort` + `ICreateRefundGatewayInput` de
  `payment-gateway.port.ts` (**mantém** `IPaymentGatewayPort` + `ICreateCheckoutGatewayInput`);
  `ICreateRefundInput` + `ICreateRefundResult` + `ICreateRefundUseCase` de `payment.input-port.ts`;
  `IRefundResult` + `RefundStatus` de `payment.model.ts` (conferido: sem outros importadores —
  `process-getnet-webhook` não usa). `IReserve.refund_status` (união literal inline) e
  `IReservePaymentMetadata` **ficam** (usados pelo webhook/review).
- **Env**: `PAGAMENTOS_API_URL` e `PAGAMENTOS_INTERNAL_API_KEY` saem de
  `agendamentos/config/env.ts` (`Env` + `loadEnv`) e de `env.spec.ts`. Refletir a remoção em
  `beach-center-server/.env.dev.example`, `beach-center-server/docker-compose.dev.yml` e
  `beach-center-server/README.md` (Princípio IV, task 003). **Não** tocar `beach-center-app`.

**DD7 — `pagamentos`: submit move para `waiting_approve` antes do upload (`PAG-mín`).**
- `SubmitManualPaymentUsecase`: após toda a validação (autorização, valor, slots) e **antes** de
  `fileStoragePort.uploadProof`, chama `updateReserveStatusPort.execute(reserve, 'waiting_approve')`
  e usa o `IReserve` retornado (`activated`) para o `prefix` do arquivo e para `reserve_number`
  no `manual_payment` (o `number` só existe **depois** dessa transição — DD3). O guard
  `reserve.status !== 'pending'` (rejeita reenvio) **fica**. As compensações em falha de
  upload/persist continuam setando `'rejected'` (de `waiting_approve` → `rejected` libera o
  `scheduling` — coerente).
- `ReviewManualPaymentUsecase`: o guard vira "aceita `reserve.status` em `['pending',
  'waiting_approve']` ou já igual ao review"; o bloco que grava `payment_method: 'PIX'` +
  `updateReserveStatusPort` roda para `pending` **ou** `waiting_approve`.
- `pagamentos/domain/models/reserva.model.ts` (cópia local): união `status` ganha
  `'waiting_approve' | 'expired'`. `IUpdateReserveStatusPort` segue o tipo (`IReserve["status"]`).

**DD8 — Autorização / rotas.** Nenhuma rota nova. `GET /reservas` continua `authMiddleware +
requireRole('ADMIN')`. `PATCH /reservas/:id/status` continua `adminOrInternalApiKey`.
`GET /ranking-agendamentos` continua `internalApiKeyMiddleware`. A varredura de expiração é
efeito colateral da listagem, não endpoint próprio.

**DD9 — `expired` como terminal defensivo no fluxo por protocolo (reserva).**
`DeleteReserveUsecase.executeByProtocol` rejeita `status === 'expired'` (409, "Reserva expirada")
além de `'cancelled'`. `FindReserveByProtocolUsecase.getCancellationInfo` retorna `can_cancel:
false` para `'expired'` (além de `'cancelled'`). O fluxo público completo por protocolo é da 006b.

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras do ecossistema** | 2 repos afetados: `beach-center-bff-agendamentos` e `beach-center-bff-pagamentos` | **CONFORME.** `agendamentos` é dono exclusivo do estado da reserva/`ranking_agendamento` e do ponto de transição; `pagamentos` é dono do comprovante e **dispara** a transição via a rota interna já existente (`PATCH /reservas/:id/status`). Esta task **remove** a única chamada `agendamentos → pagamentos` (reembolso) — a dependência entre serviços **diminui**. Nenhum uso de pattern BFF. `beach-center-app` e `beach-center-bff-campeonatos` intocados. |
| **II — Arquitetura Hexagonal / Ports** | ambos os repos já hexagonais (tasks 001/004/005) | **CONFORME.** Novos usecases de domínio (`ExpirePendingReservesUsecase`, `ExpirePendingRankingAgendamentosUsecase`) sem I/O direto — só ports injetados. Novos ports de output (`IListPendingReservesOlderThanPort`, `IListPendingRankingAgendamentosOlderThanPort`, `ISetReserveProtocolNumberPort`) com **um adapter por ação**. Geração de protocolo e regra de bloqueio por status vivem em usecase, não em adapter. Remoções (`payment-refund.port` + adapter; módulo de refund em `pagamentos`) **reduzem** superfície. `ListReservesUsecase` continua orquestrando só via ports. |
| **III — Test-First e Qualidade (NON-NEGOTIABLE)** | `coverageThreshold` global 80% nos dois repos | **CONFORME** (cobrado no `/speckit-unit-tests`). Specs a criar/alterar: máquina de estados (`waiting_approve`/`expired` bloqueiam/liberam corretamente); protocolo tardio + unicidade global + idempotência; `ExpirePendingReservesUsecase` (nada a expirar / expira e libera / não expira `< 24 h`); ordenação + `pending_approval_count` no `ListReservesUsecase`; `ExpirePendingRankingAgendamentosUsecase`; `DeleteReserveUsecase` sem refund (`refund_status = 'manual'`, mantém liberação e janela 2 h); `find-active-reserves`/`has-active-reserve` com `waiting_approve`. Em `pagamentos`: `submit` → `waiting_approve` + `prefix`/`reserve_number` pós-transição; `review` a partir de `waiting_approve`; **remoção** da suíte de refund sem quebrar `checkout`/`webhook`/`manual-payment`. ESLint estrito obrigatório. `@wip` onde aplicável. |
| **IV — Infra reproduzível com hot-reloading** | N/A (sem mudança de runtime) | **CONFORME.** Só **remoção** de 2 env vars (`PAGAMENTOS_API_URL`, `PAGAMENTOS_INTERNAL_API_KEY`) do `agendamentos` — refletida em `.env.dev.example`, `docker-compose.dev.yml` e `README.md` de `beach-center-server`. Nenhum volume/serviço novo. |

**Resultado: sem violação. Nenhum ERRO de bloqueio. Pode prosseguir para `/speckit-implement`.**

## Mapa Arquitetural (Hexagonal)

### `services/beach-center-bff-agendamentos/src/`

```
domain/models/
  reserva.model.ts                                    (ALT)  # IReserveStatus + 'waiting_approve' | 'expired'; IReserveRefundStatus + 'manual';
                                                             #   number?: string em IReserve + ICreateReserveData + IUpdateReserveData
  ranking-agendamento.model.ts                        (ALT)  # RankingAgendamentoStatus + 'expired'

domain/ports/input/
  reserva.input-port.ts                               (ALT)  # IListReservesUseCase.execute → Promise<{ data: IReserve[]; pending_approval_count: number }>

domain/ports/output/
  reserve-persistence.port.ts                         (ALT)  # + IListPendingReservesOlderThanPort; + ISetReserveProtocolNumberPort
  ranking-agendamento-persistence.port.ts             (ALT)  # + IListPendingRankingAgendamentosOlderThanPort
  payment-refund.port.ts                              (DEL)  # reembolso automático removido (DD6)

domain/usecases/reserva/
  create/create-reserve.usecase.ts                   (ALT)  # não gera number; persiste status 'pending' e number só se veio explícito
  list/list-reserves.usecase.ts                       (ALT)  # roda expiração → lista → ordena waiting_approve primeiro → { data, pending_approval_count }
  delete/delete-reserve.usecase.ts                    (ALT)  # remove refundPaymentPort + refundIfNeeded; refund_status='manual'; mantém janela 2 h e release
  find-by-protocol/find-by-protocol.usecase.ts        (ALT)  # can_cancel:false também para 'expired' (DD9)
  shared/expire-pending-reserves.usecase.ts           (NEW)  # busca pending > 24 h; delega a UpdateReserveAndSchedulingStatusUsecase('expired')

domain/usecases/ranking-agendamento/
  list/list-ranking-agendamentos.usecase.ts           (ALT)  # roda expiração antes de listar
  shared/expire-pending-ranking-agendamentos.usecase.ts (NEW) # pending > 24 h → status 'expired' + releaseEventFromSchedulings (janela exata)

domain/usecases/shared/
  update-reserve-and-scheduling-status.usecase.ts     (ALT)  # allowlist de liberação ['cancelled','rejected','expired']; waiting_approve bloqueia + revalida;
                                                             #   na transição → waiting_approve gera protocolo (IIsProtocolNumberTakenPort + ISetReserveProtocolNumberPort) se ausente

infra/schemas/
  reserva.schema.ts                                   (ALT)  # IReservaDocument.status + 'waiting_approve'|'expired'; enum status idem; enum refund_status + 'manual'
  ranking-agendamento.schema.ts                       (ALT)  # enum status + 'expired'

infra/adapters/reserva/
  list-pending-older-than/list-pending-reserves-older-than.adapter.ts   (NEW)  # find({ status:'pending', createdAt:{ $lt: cutoff } })
  set-protocol-number/set-reserve-protocol-number.adapter.ts            (NEW)  # findByIdAndUpdate({ number }) — só o campo number
  find-active-reserves-for-schedulings/find-active-reserves-for-schedulings.adapter.ts (ALT)  # status $in ['pending','waiting_approve','approved']

infra/adapters/scheduling/
  has-active-reserve/has-active-reserve-for-scheduling.adapter.ts       (ALT)  # status $in ['pending','waiting_approve','approved']

infra/adapters/ranking_agendamento/
  list-pending-older-than/list-pending-ranking-agendamentos-older-than.adapter.ts (NEW)

infra/adapters/payment/
  refund-reserve-payment.adapter.ts (+ .spec.ts)      (DEL)  # DD6

config/
  env.ts                                              (ALT)  # remove PAGAMENTOS_API_URL + PAGAMENTOS_INTERNAL_API_KEY de Env + loadEnv
  env.spec.ts                                         (ALT)  # remove asserts das 2 vars
  container.ts                                        (ALT)  # remove wiring de refund; + 3 adapters novos; injeta ExpirePendingReserves/RankingAgendamentos;
                                                             #   UpdateReserveAndSchedulingStatusUsecase recebe isProtocolNumberTakenAdapter + setReserveProtocolNumberAdapter
  container.spec.ts                                   (ALT)  # ajusta chaves/contagem

applications/controllers/reserva/
  list/list-reserves.controller.ts                    (ALT)  # responde { message, data, pending_approval_count }
```

> **Nota**: `beach-center-server/{.env.dev.example, docker-compose.dev.yml, README.md}` **(ALT)** —
> remoção das 2 env vars (Princípio IV).

### `services/beach-center-bff-pagamentos/src/`

```
domain/models/
  reserva.model.ts                                    (ALT)  # união status + 'waiting_approve' | 'expired'
  payment.model.ts                                    (ALT)  # remove IRefundResult + RefundStatus (órfãos após DD6)

domain/ports/input/
  payment.input-port.ts                               (ALT)  # remove ICreateRefundInput + ICreateRefundResult + ICreateRefundUseCase (+ import RefundStatus)

domain/ports/output/
  payment-gateway.port.ts                             (ALT)  # remove IRefundGatewayPort + ICreateRefundGatewayInput; mantém IPaymentGatewayPort + checkout

domain/usecases/manual-payment/
  submit-manual-payment.usecase.ts                    (ALT)  # → waiting_approve antes do upload; usa reserve retornado p/ prefix + reserve_number
  review-manual-payment.usecase.ts                    (ALT)  # aceita reserve.status 'waiting_approve' (além de 'pending' legado)
domain/usecases/refund/                               (DEL)  # create-refund.usecase.ts (+ .spec.ts)

applications/controllers/refund/                      (DEL)  # refund.controller.ts (+ .spec.ts)
applications/dto/create-refund.dto.ts                 (DEL)
applications/routes/routes.ts                         (ALT)  # remove POST /refunds + RefundController + refundController
applications/routes/routes.spec.ts                    (ALT)  # remove asserts da rota /refunds

infra/adapters/getnet/refund/                         (DEL)  # getnet-refund.adapter.ts + mock-refund.adapter.ts (+ .spec.ts)

config/
  container.ts                                        (ALT)  # remove MockRefundAdapter/refundGateway/createRefund + import IRefundGatewayPort
  container.spec.ts                                   (ALT)  # remove chave createRefund
  env.ts                                              (—)    # sem mudança obrigatória (getnet.refund* fica congelado; opcional documentar)

README.md                                             (ALT)  # nota: serviço legado, só PIX, comprovante + Getnet serão substituídos
```

## Checklist de Implementação

> Ordem por dependência: enums/models → ports → adapters → usecases de domínio → wiring/config →
> applications → `pagamentos`. Cada item com caminho exato. Nenhum commit (Princípio, `/speckit-implement`).

### Fase 1 — Enums e models (`agendamentos`)

- [x] `services/beach-center-bff-agendamentos/src/domain/models/reserva.model.ts` — `IReserveStatus`
      `+ 'waiting_approve' | 'expired'`; `IReserveRefundStatus` `+ 'manual'`; `number?: string` em
      `IReserve`, `ICreateReserveData`, `IUpdateReserveData`.
- [x] `services/beach-center-bff-agendamentos/src/domain/models/ranking-agendamento.model.ts` —
      `RankingAgendamentoStatus` `+ 'expired'`.
- [x] `services/beach-center-bff-agendamentos/src/infra/schemas/reserva.schema.ts` —
      `IReservaDocument.status` união literal `+ 'waiting_approve' | 'expired'`; `enum` de `status`
      idem; `enum` de `refund_status` `+ 'manual'`. `toDomainReserve` sem mudança.
- [x] `services/beach-center-bff-agendamentos/src/infra/schemas/ranking-agendamento.schema.ts` —
      `enum` de `status` `+ 'expired'`.
- [x] `tsc --noEmit` limpo após a Fase 1 (widening aditivo, sem call site novo).

### Fase 2 — Ports de output (`agendamentos`)

- [x] `.../domain/ports/output/reserve-persistence.port.ts` — `+ IListPendingReservesOlderThanPort`
      (`execute(cutoff: Date): Promise<IReserve[]>`); `+ ISetReserveProtocolNumberPort`
      (`execute(id: string, number: string): Promise<IReserve | null>`).
- [x] `.../domain/ports/output/ranking-agendamento-persistence.port.ts` —
      `+ IListPendingRankingAgendamentosOlderThanPort` (`execute(cutoff: Date): Promise<IRankingAgendamento[]>`).
- [x] `.../domain/ports/input/reserva.input-port.ts` — `IListReservesUseCase.execute` →
      `Promise<{ data: IReserve[]; pending_approval_count: number }>`.
- [x] **DELETE** `.../domain/ports/output/payment-refund.port.ts`.

### Fase 3 — Adapters (`agendamentos`)

- [x] `.../infra/adapters/reserva/list-pending-older-than/list-pending-reserves-older-than.adapter.ts`
      **(NEW)** — `ReserveModel.find({ status: 'pending', createdAt: { $lt: cutoff } })` → `toDomainReserve`.
- [x] `.../infra/adapters/reserva/set-protocol-number/set-reserve-protocol-number.adapter.ts`
      **(NEW)** — `ReserveModel.findByIdAndUpdate(id, { number }, { new: true })` → `toDomainReserve`.
- [x] `.../infra/adapters/ranking_agendamento/list-pending-older-than/list-pending-ranking-agendamentos-older-than.adapter.ts`
      **(NEW)** — `RankingAgendamentoModel.find({ status: 'pending', createdAt: { $lt: cutoff } })` → `toDomainRankingAgendamento`.
- [x] `.../infra/adapters/reserva/find-active-reserves-for-schedulings/find-active-reserves-for-schedulings.adapter.ts`
      **(ALT)** — `status: { $in: ['pending', 'waiting_approve', 'approved'] }`.
- [x] `.../infra/adapters/scheduling/has-active-reserve/has-active-reserve-for-scheduling.adapter.ts`
      **(ALT)** — `status: { $in: ['pending', 'waiting_approve', 'approved'] }`.
- [x] **DELETE** `.../infra/adapters/payment/refund-reserve-payment.adapter.ts` (+ `.spec.ts`).

### Fase 4 — Usecases de domínio (`agendamentos`)

- [x] `.../domain/usecases/reserva/create/create-reserve.usecase.ts` **(ALT)** — remove
      `customAlphabet`/`generateNumericId`; persiste `{ ...data, status: 'pending' }` (sem `number`
      forçado; passa `number: data.number` só quando presente — respeitando `exactOptionalPropertyTypes`).
- [x] `.../domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts` **(ALT)**:
  - `shouldReleaseSchedulings(status)` → `['cancelled', 'rejected', 'expired'].includes(status)`.
  - branch de revalidação: `if (status === 'pending' || status === 'approved' || status === 'waiting_approve')`;
    conjunto de "estava terminal" → `['cancelled', 'rejected', 'expired']`.
  - construtor recebe `IIsProtocolNumberTakenPort` + `ISetReserveProtocolNumberPort`.
  - antes de `updateStatusPort.execute`: `if (status === 'waiting_approve' && !reserve.number)` →
    gera protocolo (`customAlphabet('0123456789', 10)` em laço `while (await isTakenPort.execute(cand))`)
    → `await setProtocolNumberPort.execute(reserveId, cand)`; segue o fluxo com o `number` setado.
- [x] `.../domain/usecases/reserva/shared/expire-pending-reserves.usecase.ts` **(NEW)** — construtor:
      `IListPendingReservesOlderThanPort` + `UpdateReserveAndSchedulingStatusUsecase`. `execute()`:
      `cutoff = new Date(Date.now() - 24*60*60*1000)`; para cada candidato
      `await updateStatusUsecase.execute(reserve.id, 'expired')` (try/catch por item, loga e segue);
      retorna `{ expired_count }`.
- [x] `.../domain/usecases/reserva/list/list-reserves.usecase.ts` **(ALT)** — construtor:
      `IListReservesPort` + `ExpirePendingReservesUsecase`. `execute(params)`:
      `await expireUsecase.execute()` → `const data = await listPort.execute(params || {})` →
      ordena (`waiting_approve` primeiro, estável) → `pending_approval_count = data.filter(r => r.status === 'waiting_approve').length` →
      retorna `{ data, pending_approval_count }`.
- [x] `.../domain/usecases/reserva/delete/delete-reserve.usecase.ts` **(ALT)** — remove import
      `IRefundPaymentPort`, o param `refundPaymentPort`, o método `refundIfNeeded` e o bloco
      `refundData?.refund_status === 'failed'`. Novo: `const refundData = (reserve.status ===
      'approved' && reserve.total > 0) ? { refund_status: 'manual' as const } : undefined;` →
      `deleteReservePort.execute(id, refundData)`. Mantém `validateCancellationTime`, release e
      `executeByProtocol` (+ rejeita `'expired'` — DD9).
- [x] `.../domain/usecases/reserva/find-by-protocol/find-by-protocol.usecase.ts` **(ALT)** —
      `getCancellationInfo`: `if (reserveStatus === 'cancelled' || reserveStatus === 'expired' || schedulings.length === 0)`.
- [x] `.../domain/usecases/ranking-agendamento/shared/expire-pending-ranking-agendamentos.usecase.ts`
      **(NEW)** — construtor: `IListPendingRankingAgendamentosOlderThanPort` +
      `ISetRankingAgendamentoStatusPort` + `EventSchedulingImpactService`. `execute()`: `cutoff`
      idem; para cada candidato → `setStatusPort.execute(id, 'expired')` +
      `impactService.releaseEventFromSchedulings({ court, unit, specific_date: data, day_of_week:
      <derivado de data>, start_hour: hora_inicio, end_hour: hora_fim, source: 'RANKING' })`
      (mesmo shape usado pelo `delete-bulk` de ranking — conferir a assinatura real no implement).
- [x] `.../domain/usecases/ranking-agendamento/list/list-ranking-agendamentos.usecase.ts` **(ALT)** —
      construtor `+ ExpirePendingRankingAgendamentosUsecase`; `execute` roda a varredura antes do
      `listPort.execute(filter)`.

### Fase 5 — Wiring / config (`agendamentos`)

- [x] `.../config/env.ts` **(ALT)** — remove `PAGAMENTOS_API_URL` e `PAGAMENTOS_INTERNAL_API_KEY`
      de `interface Env` e de `loadEnv` (inclusive o default de `PAGAMENTOS_API_URL`).
- [x] `.../config/env.spec.ts` **(ALT)** — remove os casos que verificam as 2 vars.
- [x] `.../config/container.ts` **(ALT)**:
  - remove `import RefundReservePaymentAdapter`, `const refundReservePaymentAdapter`, e o arg no
    `new DeleteReserveUsecase(...)`.
  - `const listPendingReservesOlderThanAdapter`, `const setReserveProtocolNumberAdapter`,
    `const listPendingRankingAgendamentosOlderThanAdapter`.
  - `new UpdateReserveAndSchedulingStatusUsecase(...)` recebe `isProtocolNumberTakenAdapter` +
    `setReserveProtocolNumberAdapter` (ambos já/agora disponíveis).
  - `const expirePendingReservesUsecase = new ExpirePendingReservesUsecase(listPendingReservesOlderThanAdapter, updateReserveAndSchedulingStatusUsecase)`.
  - `new ListReservesUsecase(listReservesAdapter, expirePendingReservesUsecase)`.
  - `const expirePendingRankingAgendamentosUsecase = new ExpirePendingRankingAgendamentosUsecase(listPendingRankingAgendamentosOlderThanAdapter, setRankingAgendamentoStatusAdapter, eventSchedulingImpactService)`.
  - `new ListRankingAgendamentosUsecase(listRankingAgendamentosAdapter, expirePendingRankingAgendamentosUsecase)`.
- [x] `.../config/container.spec.ts` **(ALT)** — ajusta contagem/chaves; garante `listReserves` e
      `listRankingAgendamentos` resolvíveis; remove referências a `PAGAMENTOS_*` se houver.

### Fase 6 — Applications (`agendamentos`)

- [x] `.../applications/controllers/reserva/list/list-reserves.controller.ts` **(ALT)** —
      `const { data, pending_approval_count } = await container.listReserves.execute(filters);`
      → `res.status(200).json({ message: 'Reservas listadas com sucesso', data, pending_approval_count })`.
- [x] Conferir os outros controllers de reserva/ranking que consomem `listReserves`/
      `listRankingAgendamentos` (nenhum outro consumidor esperado — validar no implement).

### Fase 7 — `beach-center-bff-pagamentos`

- [x] `.../src/domain/models/reserva.model.ts` **(ALT)** — união `status` `+ 'waiting_approve' | 'expired'`.
- [x] `.../src/domain/usecases/manual-payment/submit-manual-payment.usecase.ts` **(ALT)** — após a
      validação e antes de `uploadProof`: `const activated = await this.updateReserveStatusPort.execute(reserve, 'waiting_approve');`
      → usar `activated.number` no `prefix` e `activated.number` como `reserve_number` no
      `manualPaymentRepositoryPort.create`. Compensações de falha continuam em `'rejected'`.
- [x] `.../src/domain/usecases/manual-payment/review-manual-payment.usecase.ts` **(ALT)** — guard:
      `if (reserve.status !== 'pending' && reserve.status !== 'waiting_approve' && reserve.status !== reserveStatus)`
      → `ConflictError`; bloco de gravação: `if (reserve.status === 'pending' || reserve.status === 'waiting_approve')`.
- [x] `.../src/domain/ports/output/payment-gateway.port.ts` **(ALT)** — remove `IRefundGatewayPort`
      + `ICreateRefundGatewayInput` (+ import `IRefundResult`); mantém checkout.
- [x] `.../src/domain/ports/input/payment.input-port.ts` **(ALT)** — remove `ICreateRefundInput` +
      `ICreateRefundResult` + `ICreateRefundUseCase` + import `RefundStatus`.
- [x] `.../src/domain/models/payment.model.ts` **(ALT)** — remove `IRefundResult` + `RefundStatus`
      (conferir no implement que não sobrou importador — `grep -r RefundStatus\|IRefundResult src/`).
- [x] **DELETE** `.../src/domain/usecases/refund/create-refund.usecase.ts` (+ `.spec.ts`).
- [x] **DELETE** `.../src/applications/controllers/refund/refund.controller.ts` (+ `.spec.ts`).
- [x] **DELETE** `.../src/applications/dto/create-refund.dto.ts`.
- [x] **DELETE** `.../src/infra/adapters/getnet/refund/getnet-refund.adapter.ts` (+ `.spec.ts`) e
      `mock-refund.adapter.ts` (+ `.spec.ts`).
- [x] `.../src/applications/routes/routes.ts` **(ALT)** — remove `import RefundController`, `const
      refundController`, e a linha `routes.post('/refunds', ...)`.
- [x] `.../src/applications/routes/routes.spec.ts` **(ALT)** — remove asserts de `/refunds`.
- [x] `.../src/config/container.ts` **(ALT)** — remove `import MockRefundAdapter`, `import
      { IRefundGatewayPort }` (ou ajusta o import conjunto de `payment-gateway.port`), `const
      refundGateway`, `createRefund: new CreateRefundUsecase(...)` e `import CreateRefundUsecase`.
- [x] `.../src/config/container.spec.ts` **(ALT)** — remove a chave `createRefund`.
- [x] `.../README.md` **(ALT)** — parágrafo: serviço **legado**; pagamento **só PIX**; inserção/
      validação de comprovante e o restante da integração Getnet serão migrados para
      `beach-center-bff-injection` / `beach-center-bff-llm-engine` numa task futura (ver
      `tasks/006a-.../context.md` › "Sistema de pagamento — futuro").

### Fase 8 — Infra (`beach-center-server`)

- [x] `beach-center-server/.env.dev.example` **(ALT)** — remove `PAGAMENTOS_API_URL` /
      `PAGAMENTOS_INTERNAL_API_KEY` da seção de `agendamentos` (manter as de `pagamentos` se
      existirem para o próprio serviço).
- [x] `beach-center-server/docker-compose.dev.yml` **(ALT)** — remove as 2 vars do `environment`
      do serviço `agendamentos`.
- [x] `beach-center-server/README.md` **(ALT)** — remove as 2 vars da tabela/lista de env do
      `agendamentos`.

### Fase 9 — Verificação local (sem commit)

- [x] `agendamentos`: `npx tsc --noEmit` limpo; `npx eslint` limpo nos arquivos tocados.
- [x] `pagamentos`: `npx tsc --noEmit` limpo; `npx eslint` limpo nos arquivos tocados.
- [x] Suíte existente dos dois repos executada — anotar specs que quebram por mudança de contrato
      (esperado: `list-reserves`, `update-reserve-and-scheduling-status`, `delete-reserve`,
      `create-reserve`, `env`, `container` em `agendamentos`; `submit`/`review`/`routes`/
      `container` + suíte de refund em `pagamentos`). O **ajuste/criação** desses specs é do
      `/speckit-unit-tests`; aqui só se registra o diff de contrato.

## Desvios / notas de implementação (`/speckit-implement`, 2026-09-10)

1. **DTOs de status alargados** (não estavam no Mapa, exigidos por DD1/DD7/AC-2). `yup .oneOf`:
   - `applications/dto/update-status.dto.ts` (`PATCH /reservas/:id/status`, a rota que `pagamentos`
     chama) — `+ 'waiting_approve' | 'expired'`. **Sem isso o `submit` de `pagamentos` tomaria 400.**
   - `applications/dto/update-reserva.dto.ts` (`PATCH /reservas/:id` admin) — mesma lista completa.
   - `applications/dto/update-payment-metadata.dto.ts` — `refund_status` `+ 'manual'`.
2. **`domain/errors.ts` → `ICloseDateReserveConflict.number`** virou `string | undefined` (efeito do
   `IReserve.number` opcional — `close-date.usecase.ts` mapeia reservas em conflito e uma reserva
   `pending` pode não ter protocolo).
3. **`domain/usecases/reserva/update/update-reserve.usecase.ts`** (não listado no Mapa): o branch
   bloqueia/libera `scheduling` usava `status !== 'cancelled' && status !== 'rejected'`. Passou a
   usar a allowlist `RELEASING_STATUSES = ['cancelled', 'rejected', 'expired']` para `expired`
   liberar corretamente (coerência com DD2).
4. **`config/container.ts` (agendamentos)** — `listReservesUsecase` foi movido para **depois** da
   seção 5 (precisa do `updateReserveAndSchedulingStatusUsecase`, via `ExpirePendingReservesUsecase`).
5. **`beach-center-server/README.md`** — nenhuma alteração necessária: só continha
   `VITE_PAGAMENTOS_API_URL` (var do front, fora de escopo), não as 2 vars de `agendamentos`.
   `.env.dev.example` — removida a linha comentada `PAGAMENTOS_INTERNAL_API_KEY` e reescrito o
   comentário do bloco de chaves internas (agendamentos não fala mais com pagamentos).
6. **`pagamentos`**: `authOrInternalApiKey` / `internalApiKeyMiddleware` continuam exportados e
   testados, mas nenhuma rota os usa mais (só o `/refunds` usava). `env.getnet.refundPathTemplate`
   / `refundMethod` ficaram órfãos — **mantidos** (config Getnet congelada, fora de escopo).
7. **Specs quebrados deixados para `/speckit-unit-tests`** (7 suítes / 21 testes em `agendamentos`,
   todos por mudança de contrato desta task): `create-reserve.usecase`, `delete-reserve.usecase`,
   `list-reserves.usecase`, `list-reserves.controller`, `list-ranking-agendamentos.usecase`,
   `find-active-reserves-for-schedulings.adapter`, `has-active-reserve-for-scheduling.adapter`.
   Specs de config já ajustados (`env.spec`, `container.spec` x2, `routes.spec` de `pagamentos`).
   **`pagamentos`: suíte 33/33 verde** (submit/review specs seguem passando).

## Testes unitários (`/speckit-unit-tests`, 2026-09-10)

Todos os specs afetados corrigidos + specs novos para os 5 arquivos novos + cobertura dos `AC-*`.

**`agendamentos`** — 279 suítes / 1316 testes verdes · cobertura **98.87 % stmt / 92.7 % branch**.
- Reescritos: `list-reserves.usecase.spec` (`{ data, pending_approval_count }` + ordenação +
  varredura antes de listar), `delete-reserve.usecase.spec` (sem port de refund; `refund_status
  = 'manual'`; `expired` terminal no protocolo), `list-reserves.controller.spec` (resposta nova),
  `list-ranking-agendamentos.usecase.spec` (expiração antes de listar).
- Ajustados: `find-active-reserves-for-schedulings.adapter.spec` / `has-active-reserve-for-scheduling.adapter.spec`
  (`+ waiting_approve`), `create-reserve.usecase.spec` (sem protocolo na criação),
  `create-reserve.adapter.spec` (omite `number`), `find-by-protocol.usecase.spec` (`expired`),
  `update-reserve.usecase.spec` (`expired` libera), `update-reserve-and-scheduling-status.usecase.spec`
  (2 ports novos + `waiting_approve` bloqueia + `expired` libera + protocolo tardio/idempotente).
- Novos: `expire-pending-reserves.usecase.spec`, `expire-pending-ranking-agendamentos.usecase.spec`,
  `list-pending-reserves-older-than.adapter.spec`, `set-reserve-protocol-number.adapter.spec`,
  `list-pending-ranking-agendamentos-older-than.adapter.spec` — todos os arquivos novos a 100 %.

**`pagamentos`** — 33 suítes / 147 testes verdes · cobertura **98.19 % stmt / 81.37 % branch**.
- `submit-manual-payment.usecase.spec`: `→ waiting_approve` antes do upload; `prefix`/`reserve_number`
  vêm do `number` retornado pela transição (AC-21).
- `review-manual-payment.usecase.spec`: revisa a partir de `waiting_approve` (AC-24/AC-25) e segue
  revisando reserva legada `pending` (AC-26).
- Suíte de refund removida junto com o módulo — `checkout`/`webhook` intactos (AC-29).

`tsc --noEmit` e `eslint` limpos nos dois repos. Nenhum commit.

## Validação (`/speckit-validate`, 2026-09-10 — autônoma, a pedido da usuária)

Revisão arquivo por arquivo dos 3 repos (produção → testes). Correções aplicadas:
- `domain/models/reserva.model.ts` — doc-comment de `IRefundOutcome` reescrito (não cita mais o
  `payment-refund port` removido).
- `domain/models/ranking-agendamento.model.ts` — doc-comment de `RankingAgendamentoStatus`
  atualizado (o motivo "alargar quebra o schema de reserva" não vale mais — a 006a alargou
  `IReserveStatus` model + schema juntos).
Nenhum defeito funcional encontrado. `tsc`, `eslint` e as suítes (279+33 / 1316+147) verdes nos
dois serviços. Alterações **staged** nos 3 repos, sem commit.

## Testes de componente (`/speckit-component-tests`, 2026-09-10)

**Não aplicável.** A task 006a é 100 % backend (`beach-center-bff-agendamentos` +
`beach-center-bff-pagamentos`); `beach-center-app` **não foi tocado** e não possui Cypress/Cucumber
configurado (nenhuma das tasks 001–005, também backend, gerou `.feature`). Não há superfície de
UI/E2E para cobrir. A validação dos fluxos fica pelos testes unitários (acima) e pelos cenários
exploratórios do `/speckit-test`.

## Critérios de Aceite (formais)

> Formato Given / When / Then. Base para `/speckit-unit-tests`, `/speckit-component-tests` e
> `/speckit-test`. "reserva" = reserva comum de aluguel; "RA" = `ranking_agendamento`.

### Máquina de estados — reserva

**AC-1 — Reserva nasce `pending` e sem protocolo.**
- **Given** um payload válido de criação de reserva sem `number`
- **When** `POST /reservas`
- **Then** a reserva é persistida com `status === 'pending'` e **sem** `number` (campo ausente),
  e os `scheduling`(s) vinculados ficam indisponíveis.

**AC-2 — Comprovante gera o protocolo na transição para `waiting_approve`.**
- **Given** uma reserva `pending` sem `number`
- **When** o status é alterado para `'waiting_approve'` via `PATCH /reservas/:id/status`
- **Then** a reserva passa a `status === 'waiting_approve'`, recebe um `number` de 10 dígitos
  **único globalmente** (não colide com `reservas.number` nem `ranking_agendamentos.numero_protocolo`),
  e os `scheduling`(s) **permanecem** indisponíveis.

**AC-3 — Protocolo é idempotente.**
- **Given** uma reserva que já possui `number` (ex.: reprocessamento)
- **When** o status é alterado para `'waiting_approve'`
- **Then** o `number` **não** é regenerado nem alterado.

**AC-4 — `waiting_approve` bloqueia o agendamento para novas reservas/eventos.**
- **Given** um `scheduling` cuja única reserva está `waiting_approve`
- **When** o sistema avalia disponibilidade / conflito desse `scheduling` (nova reserva,
  `close-date`, bloqueio de campeonato/ranking)
- **Then** o `scheduling` é tratado como **ocupado** (reserva ativa presente).

**AC-5 — Aprovação.**
- **Given** uma reserva `waiting_approve`
- **When** o status vai para `'approved'`
- **Then** `status === 'approved'` e os `scheduling`(s) seguem indisponíveis.

**AC-6 — Rejeição libera o agendamento.**
- **Given** uma reserva `waiting_approve`
- **When** o status vai para `'rejected'`
- **Then** `status === 'rejected'` e cada `scheduling` volta a **disponível** se nenhuma outra
  reserva ativa / evento confirmado o bloquear.

### Expiração — reserva

**AC-7 — Reserva `pending` há mais de 24 h expira na listagem.**
- **Given** uma reserva `pending` com `createdAt` anterior a `agora − 24 h` e sem comprovante
- **When** `GET /reservas` é chamado
- **Then** antes de responder, a reserva passa a `status === 'expired'` e seus `scheduling`(s)
  são liberados (se nada mais os bloquear); a reserva aparece na resposta já como `expired`.

**AC-8 — Reserva `pending` recente NÃO expira.**
- **Given** uma reserva `pending` com `createdAt` de menos de 24 h atrás
- **When** `GET /reservas`
- **Then** a reserva continua `pending` e seus `scheduling`(s) seguem bloqueados.

**AC-9 — Expiração não afeta outros status.**
- **Given** reservas em `waiting_approve` / `approved` / `cancelled` / `rejected` com `createdAt`
  antigo
- **When** `GET /reservas`
- **Then** nenhuma dessas tem o status alterado pela varredura.

**AC-10 — `expired` é terminal.**
- **Given** uma reserva `expired`
- **When** tenta-se cancelá-la por protocolo (`executeByProtocol`)
- **Then** retorna `409` ("Reserva expirada") e nada é alterado.

### Listagem — reserva

**AC-11 — `waiting_approve` vem primeiro.**
- **Given** reservas em `pending`, `approved` e `waiting_approve`
- **When** `GET /reservas`
- **Then** todas as `waiting_approve` aparecem antes das demais no array `data`; a ordem relativa
  das não-`waiting_approve` é preservada.

**AC-12 — Contagem de pendentes de aprovação.**
- **Given** N reservas em `waiting_approve` (após a varredura de expiração)
- **When** `GET /reservas`
- **Then** a resposta inclui `pending_approval_count === N`, e `data` continua sendo o array de
  reservas.

**AC-13 — Filtros continuam funcionando.**
- **Given** `GET /reservas?status=approved` (ou por `name`/`email`/`phone`/`scheduling_id`)
- **When** a chamada é feita
- **Then** o filtro é aplicado normalmente; a varredura de expiração roda mesmo assim; a contagem
  reflete os `waiting_approve` do resultado filtrado.

### Cancelamento sem reembolso — reserva

**AC-14 — Cancelar reserva aprovada marca reembolso manual.**
- **Given** uma reserva `approved` com `total > 0`, dentro da janela de 2 h
- **When** `PATCH /reservas/:id/delete`
- **Then** a reserva é cancelada (`status === 'cancelled'`), `refund_status === 'manual'`,
  **nenhuma** chamada HTTP a `pagamentos` é feita, e os `scheduling`(s) são liberados.

**AC-15 — Cancelar reserva sem pagamento não seta `refund_status`.**
- **Given** uma reserva `pending`/`waiting_approve` (ou `approved` com `total === 0`)
- **When** `PATCH /reservas/:id/delete`
- **Then** a reserva é cancelada e `refund_status` permanece ausente/inalterado; sem chamada a
  `pagamentos`.

**AC-16 — Janela de 2 h preservada.**
- **Given** uma reserva cujo primeiro `scheduling` começa em menos de 2 h
- **When** `PATCH /reservas/:id/delete` sem `skipCancellationTimeValidation`
- **Then** retorna `400` ("...ate 2 horas antes...") e nada é alterado.

**AC-17 — Nenhum caminho de código de reembolso automático permanece.**
- **Given** o código de `agendamentos` e `pagamentos` após a task
- **When** se busca por `payment-refund`, `RefundReservePayment`, `POST /refunds`,
  `IRefundGatewayPort`, `create-refund`
- **Then** não há nenhum arquivo, rota, porta, adapter ou wiring de reembolso automático; o
  checkout Getnet e o processamento de webhook seguem intactos.

### Expiração — `ranking_agendamento`

**AC-18 — RA `pending` há mais de 24 h expira ao listar.**
- **Given** um RA `pending` com `createdAt` anterior a `agora − 24 h`
- **When** `GET /ranking-agendamentos` (rota interna)
- **Then** antes de responder, o RA passa a `status === 'expired'` e o bloqueio da janela exata
  (`hora_inicio`–`hora_fim`, aquela `data`, aquela quadra/unidade) é liberado nos `scheduling`(s)
  correspondentes (se nada mais os bloquear).

**AC-19 — RA `pending` recente / outros status não expiram.**
- **Given** um RA `pending` com menos de 24 h, e RAs em `waiting_approve`/`approved`/`cancelled`
- **When** `GET /ranking-agendamentos`
- **Then** nenhum tem o status alterado pela varredura.

**AC-20 — RA em `waiting_approve` (comprovante já enviado) nunca expira.**
- **Given** um RA que recebeu comprovante e está `waiting_approve` há vários dias
- **When** `GET /ranking-agendamentos`
- **Then** continua `waiting_approve`.

### `beach-center-bff-pagamentos` — comprovante

**AC-21 — Enviar comprovante move a reserva para `waiting_approve` antes do upload.**
- **Given** uma reserva `pending` sem `number` e um comprovante válido (valor e slots corretos)
- **When** `POST /public/transaction-history` (ou a rota autenticada)
- **Then** a reserva é transicionada para `waiting_approve` (gerando o `number` em `agendamentos`),
  o arquivo é armazenado com `prefix` igual ao **novo** `number`, e o `manual_payment` criado
  guarda `reserve_number` igual ao novo `number` e `status === 'pending'`.

**AC-22 — Falha no upload compensa para `rejected`.**
- **Given** o cenário do AC-21, mas o `fileStoragePort.uploadProof` lança
- **When** o usecase trata o erro
- **Then** a reserva é setada para `'rejected'` e o erro é propagado; nenhum `manual_payment` é
  persistido.

**AC-23 — Reenvio de comprovante é bloqueado.**
- **Given** uma reserva que já não está `pending` (`waiting_approve`/`approved`/...)
- **When** `POST .../transaction-history`
- **Then** retorna `409` ("A reserva nao esta pendente de pagamento"); nada é alterado.

**AC-24 — Revisão de comprovante a partir de `waiting_approve`.**
- **Given** um `manual_payment` `pending` cuja reserva está `waiting_approve`
- **When** `PATCH /transaction-history/:id/review` com `status: 'approved'`
- **Then** a reserva recebe `payment_method: 'PIX'`, é transicionada para `'approved'` via
  `PATCH {agendamentosApiUrl}/reservas/:id/status`, e o `manual_payment` fica `approved`.

**AC-25 — Revisão rejeitando.**
- **Given** o mesmo cenário do AC-24
- **When** review com `status: 'rejected'`
- **Then** a reserva vai para `'rejected'` (liberando o `scheduling`) e o `manual_payment` fica
  `rejected`.

**AC-26 — Revisão de reserva legada `pending` continua funcionando.**
- **Given** um `manual_payment` `pending` cuja reserva ainda está `pending` (dado legado,
  pré-migração)
- **When** review `approved`/`rejected`
- **Then** o comportamento anterior é mantido (transição direta a partir de `pending`).

### Qualidade / regressão

**AC-27 — `tsc` e ESLint limpos.**
- **Given** os dois repositórios após a implementação
- **When** `tsc --noEmit` e `eslint` rodam
- **Then** zero erros em ambos.

**AC-28 — Env sem as variáveis de reembolso.**
- **Given** `agendamentos/config/env.ts` e a infra de `beach-center-server`
- **When** o serviço sobe sem `PAGAMENTOS_API_URL` / `PAGAMENTOS_INTERNAL_API_KEY` definidas
- **Then** o boot ocorre normalmente (as vars não são mais lidas nem exigidas).

**AC-29 — Checkout e webhook de `pagamentos` intactos.**
- **Given** a suíte de `pagamentos` após a remoção do módulo de reembolso
- **When** os testes de `create-checkout`, `process-getnet-webhook`, `submit`/`review` e
  `list-payment-methods` rodam
- **Then** todos passam (a remoção do refund não os afeta).

## Próximo passo sugerido

`/speckit-implement` — implementa o checklist acima (sem commit), com conformidade estrita ao
ESLint. Depois: `/speckit-unit-tests`, `/speckit-component-tests`, `/speckit-validate`,
`/speckit-test`, `/speckit-complete`, `/speckit-documentation`. Em seguida, a **task 006b**
(troca de dia + cancelamento público por protocolo + janela de 2 h nas rotas internas da 005 +
auditoria de reagendamento + WhatsApp), que depende desta.
