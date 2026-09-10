# Testes Exploratórios Manuais — Task 006a (Ciclo de vida do pagamento)

> Gerado por `/speckit-test`. Roteiro de QA manual (o projeto não tem analistas de QA dedicados).
> **Não** contém código de teste automatizado — esses estão em `/speckit-unit-tests`
> (`agendamentos` 279 suítes / 1316 testes; `pagamentos` 33 / 147).
> Base: `plan.md` (Critérios de Aceite **AC-1 … AC-29**) + `context.md` (decisões A1/D2/D4/D5/D6/D-exp/B-list/PIX/PAG-mín).

---

## 1. Escopo e pré-condições

### 1.1 Serviços envolvidos

| Serviço | Papel no teste | Auth | Base URL |
|---|---|---|---|
| `beach-center-bff-agendamentos` | **Dono do estado** da reserva e do `ranking_agendamento`. Máquina de estados, protocolo tardio, expiração lazy, listagem ordenada, cancelamento sem reembolso. | `authMiddleware` + `requireRole('ADMIN')` (Firebase) nas rotas de admin; `adminOrInternalApiKey` no `PATCH /reservas/:id/status`; `internalApiKeyMiddleware` em `/ranking-agendamentos`; rotas públicas por protocolo sem auth | `http://localhost:<PORT_AGENDAMENTOS>/api/v1` |
| `beach-center-bff-pagamentos` | Recebe o comprovante e **dispara** a transição de status. Legado; só PIX. | `authMiddleware` / público (`/public/transaction-history`) / `requireRole('ADMIN')` no review | `http://localhost:<PORT_PAGAMENTOS>/api/v1` |
| `beach-center-bff-usuarios` | Emissão do token ADMIN. | — | — |
| `beach-center-bff-campeonatos` | **Só regressão** — chama `POST /ranking-agendamentos` (create-bulk) e não muda de contrato nesta task. | — | — |
| ~~`beach-center-app`~~ | **Fora de escopo** — nenhuma tela foi tocada. | — | — |

> ⚠️ **O que mudou vs. antes desta task:**
> - Reserva **nasce `pending` sem `number`**. O protocolo (`number`, 10 dígitos) é gerado **só** na
>   transição `→ waiting_approve` (envio do comprovante), com unicidade global (reservas +
>   `ranking_agendamentos`).
> - Novos status: `IReserveStatus` ganhou `'waiting_approve'` e `'expired'`;
>   `RankingAgendamentoStatus` ganhou `'expired'`.
> - `GET /reservas` agora responde `{ message, data, pending_approval_count }` (era `{ message, data }`),
>   roda a **varredura de expiração** antes de listar e devolve os `waiting_approve` **primeiro**.
> - `GET /ranking-agendamentos` (interno) roda a varredura de expiração de `ranking_agendamento`
>   `pending` antes de listar.
> - **Reembolso automático removido.** `agendamentos` **não chama mais `pagamentos`**. Cancelar
>   reserva `approved` grava `refund_status: 'manual'` — nenhuma chamada HTTP externa.
> - Rota `POST /refunds` de `pagamentos` **não existe mais**.

### 1.2 Ambiente

- Subir o ecossistema com `docker-compose.dev.yml` (task 003) ou `npm run dev` em cada serviço.
- Mongo limpo (ou base de staging isolada) — vários cenários mexem em reservas e liberação de quadra.
- Variáveis obrigatórias:
  - `agendamentos`: `DB`, `FIREBASE_PROJECT_ID`. **Não** precisa mais de `PAGAMENTOS_API_URL` /
    `PAGAMENTOS_INTERNAL_API_KEY` (AC-28 — o serviço deve subir sem elas).
  - `pagamentos`: `DB`, `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY` (para o review
    conseguir chamar `PATCH /reservas/:id/status`), `PAYMENT_PROVIDER` (`mock` ou `getnet`).
- Variáveis opcionais (comprovante): `GOOGLE_DRIVE_*` em `pagamentos` — testar os dois modos
  (upload real vs. fallback base64).
- **Manipulação de tempo** (expiração 24 h): como não há cron, o teste da varredura precisa de
  documentos com `createdAt` "envelhecido". Opções:
  1. Inserir a reserva/`ranking_agendamento` direto no Mongo com `createdAt` de 25 h atrás
     (`db.reservas.updateOne({_id}, {$set:{createdAt: new Date(Date.now()-25*3600*1000)}})`).
  2. Ou adiantar o relógio do container.

### 1.3 Dados de seed

| Item | Detalhe |
|---|---|
| Unidade + quadra | 1 `unit` + 1 `court` ativos. |
| Agendamentos (`scheduling`) | Pelo menos 3 slots **futuros** e disponíveis na mesma quadra/data (ex.: 08:00, 09:00, 10:00), 1 slot começando **em < 2 h** (para a janela de cancelamento). |
| Usuário ADMIN | Token Firebase válido com `role=ADMIN`. |
| Usuário consumidor | E-mail que baterá com `reserve.email` (para autorização do `submit`). |
| Preços | Regra de `pagamentos`: `EXPECTED_AMOUNT_BY_DURATION = {1:80, 2:120, 3:160}` — a reserva de N slots tem `total` = tabela. |
| `ranking_agendamento` | 1 criado via `POST /ranking-agendamentos` (create-bulk, `x-api-key`) — nasce `pending`, com `numero_protocolo`, bloqueando o slot. |

### 1.4 Perfis / permissões

| Ação | Quem |
|---|---|
| `GET /reservas`, `PATCH /reservas/:id`, `PATCH /reservas/:id/delete` | ADMIN (Firebase) |
| `PATCH /reservas/:id/status` | ADMIN **ou** `x-api-key` interna (é por aqui que `pagamentos` transita o status) |
| `GET/PATCH /ranking-agendamentos*` (exceto comprovante) | `x-api-key` interna |
| `PATCH /ranking-agendamentos/protocolo/:n/comprovante` | **público** (sem auth) |
| `POST /public/transaction-history` (envio de comprovante) | público (autoriza por `requesterEmail` == `reserve.email` ou token de link público) |
| `PATCH /transaction-history/:id/review` | ADMIN |

---

## 2. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **H-1** (AC-1) | Slots S1 futuros e disponíveis | `POST /reservas` com payload válido, **sem** `number` | `201`. `GET /reservas/:id` → `status: "pending"`, **sem** campo `number`. Slots de S1 ficam `available: false`. |
| **H-2** (AC-2, AC-4, AC-21) | Reserva R1 `pending` de H-1; `total` = tabela; `requesterEmail` = `R1.email` | `POST /public/transaction-history` com `reserve_id=R1`, `amount`/`slots` corretos, `proof_file` válido | `200`/`201`. `GET /reservas/:R1` → `status: "waiting_approve"`, `number` com **10 dígitos**. Slots continuam `available: false`. O `manual_payment` guarda `reserve_number` == `number` gerado; o arquivo no Drive/base64 tem `prefix` == `number` gerado. |
| **H-3** (AC-2 unicidade) | Existe uma reserva/`ranking_agendamento` com `number`/`numero_protocolo` = `X` | Repetir H-2 várias vezes (várias reservas) | Nenhum `number` gerado colide com `X` nem entre si (consulta global em `reservas` + `ranking_agendamentos`). |
| **H-4** (AC-5) | Reserva R1 `waiting_approve` + `manual_payment` `pending` | ADMIN: `PATCH /transaction-history/:id/review` com `status: "approved"` | `200`. `R1.status: "approved"`, `R1.payment_method: "PIX"`. Slots seguem `available: false`. `manual_payment.status: "approved"`. |
| **H-5** (AC-6) | Reserva R1 `waiting_approve` + `manual_payment` `pending` | ADMIN: review com `status: "rejected"` | `200`. `R1.status: "rejected"`. Slots de R1 voltam a `available: true` (se nada mais os bloquear). `manual_payment.status: "rejected"`. |
| **H-6** (AC-11, AC-12) | 5 reservas: 2 `waiting_approve` (Wa, Wb — criadas nesta ordem), 1 `pending`, 1 `approved`, 1 `cancelled` | ADMIN: `GET /reservas` | `200` com `{ data, pending_approval_count }`. `data[0]` e `data[1]` são `Wa` e `Wb` (nessa ordem); as demais mantêm a ordem do banco. `pending_approval_count === 2`. |
| **H-7** (AC-13) | Cenário de H-6 | `GET /reservas?status=waiting_approve` | Só as 2 `waiting_approve`; `pending_approval_count === 2`. `GET /reservas?status=approved` → só a `approved`; `pending_approval_count === 0`. |
| **H-8** (AC-7) | Reserva R2 `pending` **sem comprovante**, com `createdAt` de 25 h atrás (ver §1.2); slots de R2 bloqueados | ADMIN: `GET /reservas` | Antes de responder: `R2.status` vira `"expired"`; slots de R2 voltam a `available: true` (se nada mais bloquear). `R2` aparece na resposta já como `expired`. |
| **H-9** (AC-14) | Reserva R3 `approved`, `total > 0`, primeiro slot começa em > 2 h | ADMIN: `PATCH /reservas/:R3/delete` | `200`. `R3.status: "cancelled"`, `R3.refund_status: "manual"`. **Nenhuma** requisição HTTP sai de `agendamentos` para `pagamentos` (checar logs / mock de rede). Slots de R3 liberados. |
| **H-10** (AC-15) | Reserva R4 `pending` (ou `approved` com `total = 0`), dentro da janela | ADMIN: `PATCH /reservas/:R4/delete` | `200`. `R4.status: "cancelled"`, `refund_status` **ausente/inalterado**. Sem chamada a `pagamentos`. |
| **H-11** (AC-18) | `ranking_agendamento` RA1 `pending` (nunca recebeu comprovante), `createdAt` 25 h atrás, bloqueando o slot | Chamar `GET /ranking-agendamentos` (`x-api-key`) | Antes de responder: `RA1.status: "expired"`; o bloqueio da janela exata (`hora_inicio`–`hora_fim`, aquela `data`, aquela quadra) é liberado nos `scheduling`s (se nada mais os bloquear). |
| **H-12** (AC-24) | `manual_payment` `pending` cuja reserva está `waiting_approve` | ADMIN: review `approved` | Como H-4 — a revisão parte de `waiting_approve` sem erro. |
| **H-13** (AC-26 — legado) | Inserir manualmente uma reserva `pending` **com** `number` + um `manual_payment` `pending` apontando pra ela (simula dado pré-migração) | ADMIN: review `approved` | `200`. Reserva → `approved`, `payment_method: "PIX"`. Comportamento anterior preservado. |
| **H-14** (AC-29 — regressão de checkout) | `PAYMENT_PROVIDER=mock` | `POST /checkout` autenticado com payload válido | `200` — fluxo de checkout Getnet/mock **intacto** após a remoção do módulo de reembolso. |

---

## 3. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **E-1** (AC-23) | Reserva R1 já `waiting_approve` (ou `approved`/`rejected`/`cancelled`/`expired`) | `POST /public/transaction-history` de novo para R1 | `409 "A reserva nao esta pendente de pagamento"`. Nada muda. |
| **E-2** (AC-22) | Reserva `pending`; `GOOGLE_DRIVE_*` apontando para credencial inválida (forçar erro de upload) | `POST /public/transaction-history` válido | Erro propagado (5xx). `GET /reservas/:id` → `status: "rejected"`. **Nenhum** `manual_payment` persistido. Slots liberados (via `rejected`). |
| **E-3** (AC-22 — persistência) | Reserva `pending`; derrubar o Mongo de `pagamentos` logo após o upload (ou simular) | `POST /public/transaction-history` | Erro propagado. Reserva compensada para `rejected`. |
| **E-4** (AC-10) | Reserva `expired` (de H-8) com `number` (chegou a ter comprovante antes de expirar? use uma que expirou ainda em `pending` — não tem `number`; para este caso force via Mongo um `number` + `status: expired`) | `PATCH /reservas/protocolo/:number/cancel` (rota pública de cancelamento por protocolo) | `409 "Reserva expirada"`. Nada muda. |
| **E-5** (AC-10 — cancelada) | Reserva `cancelled` com `number` | `PATCH /reservas/protocolo/:number/cancel` | `409 "Reserva ja foi cancelada"` (comportamento anterior, não regrediu). |
| **E-6** (AC-16) | Reserva `approved`, primeiro slot começa **em < 2 h** | ADMIN: `PATCH /reservas/:id/delete` | `400 "Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"`. `refund_status` **não** é setado. Status inalterado. |
| **E-7** (AC-17) | — | `POST {PAGAMENTOS_API_URL}/refunds` (com qualquer payload) | `404` (rota não existe mais). `grep -r "POST /refunds\|payment-refund\|IRefundGatewayPort\|create-refund" services/` → **zero** resultados em código de produção. |
| **E-8** (AC-27) | — | `npx tsc --noEmit` e `npx eslint src/` em `agendamentos` **e** `pagamentos` | Zero erros nos dois. |
| **E-9** (AC-28) | Remover `PAGAMENTOS_API_URL` e `PAGAMENTOS_INTERNAL_API_KEY` do ambiente de `agendamentos` | `npm run dev` / subir o container | Boot normal, sem erro de "variável não configurada". `GET /api/v1` responde. |
| **E-10** (transição inválida via API) | Reserva `pending` | `PATCH /reservas/:id/status` com `{ "status": "banana" }` | `400` (DTO `yup .oneOf`). Com `{ "status": "waiting_approve" }` → `200` e gera protocolo (é a rota que `pagamentos` usa). |
| **E-11** (auth do review) | `manual_payment` `pending` | `PATCH /transaction-history/:id/review` **sem** token ADMIN | `401`/`403`. Reserva e comprovante inalterados. |
| **E-12** (submit não autorizado) | Reserva `pending` | `POST /public/transaction-history` com `requesterEmail` **diferente** de `reserve.email` e sem `public_reserve_token` | `403 "Acesso negado para esta reserva"`. Status inalterado (não vira `waiting_approve`). |
| **E-13** (valor divergente) | Reserva `pending`, `total = 80` | `POST /public/transaction-history` com `amount = 999` | `400 "Os dados do pagamento nao correspondem a reserva"`. Reserva continua `pending` (não transita). |
| **E-14** (falha na transição de status durante o submit) | Reserva `pending`; tornar um slot da reserva **passado** logo antes do submit | `POST /public/transaction-history` válido | A chamada `→ waiting_approve` falha (validação de "agendamento passado" em `agendamentos`) → erro propagado; `manual_payment` **não** é criado; reserva permanece `pending` (pode ter recebido `number` sem mudar de status — ver E-15). |

---

## 4. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **X-1** (AC-3 idempotência) | Reserva `pending` que **já** tem `number` (ex.: legado, ou E-14 gerou o `number` sem mudar status) | `PATCH /reservas/:id/status` `waiting_approve` | O `number` **não** é regenerado. Transita para `waiting_approve`. |
| **X-2** (AC-8) | Reserva `pending` com `createdAt` de **23 h 30 min** atrás | `GET /reservas` | Continua `pending`. Slots seguem bloqueados. (Fronteira exata dos 24 h.) |
| **X-3** (AC-9) | Reservas antigas (`createdAt` > 24 h) em `waiting_approve`, `approved`, `cancelled`, `rejected` | `GET /reservas` | Nenhuma muda de status pela varredura. Só `pending` é alvo. |
| **X-4** (AC-20) | `ranking_agendamento` que **recebeu comprovante** e está `waiting_approve` há vários dias | `GET /ranking-agendamentos` | Continua `waiting_approve` — a varredura só pega `status: 'pending'`. |
| **X-5** (AC-19) | `ranking_agendamento` `pending` com `createdAt` de 10 h atrás | `GET /ranking-agendamentos` | Continua `pending`; bloqueio mantido. |
| **X-6** (expiração parcial) | 3 reservas `pending` antigas; forçar erro na 2ª (ex.: apagar um `scheduling` vinculado a ela) | `GET /reservas` | A 1ª e a 3ª expiram; a 2ª é pulada (erro logado, não interrompe a listagem). Resposta ainda `200`. |
| **X-7** (bloqueio de quadra por `waiting_approve` — AC-4) | Slot S ocupado por reserva `waiting_approve`. Tentar criar **outra** reserva no mesmo S; e tentar bloquear S por `close-date` / campeonato / ranking | Todas as tentativas tratam S como **ocupado** (há reserva ativa). `find-active-reserves` e `has-active-reserve` consideram `waiting_approve`. |
| **X-8** (contagem após expiração — B-list) | Reservas: 1 `waiting_approve`, 1 `pending` antiga (vai expirar) | `GET /reservas` | A varredura roda **antes** da contagem: `pending_approval_count` reflete só o(s) `waiting_approve` remanescente(s); a expirada não conta e aparece como `expired`. |
| **X-9** (ordenação estável) | 4 reservas não-`waiting_approve` (A, B, C, D nessa ordem no banco) + 0 `waiting_approve` | `GET /reservas` | `data` = `[A, B, C, D]` — ordem do banco preservada; `pending_approval_count === 0`. |
| **X-10** (total zero / negativo) | Reserva `approved` com `total = 0`; e uma com `total` negativo (se o schema permitir) | `PATCH /reservas/:id/delete` | `total <= 0` → **não** seta `refund_status`. Cancela normalmente. |
| **X-11** (fuso / virada de dia na expiração) | Reserva criada 23:30 (horário local) ontem, agora 00:15 hoje (< 24 h de diferença real) | `GET /reservas` | **Não** expira — o corte usa `Date.now() - 24h` em UTC, não "dia calendário". |
| **X-12** (reserva multi-slot) | Reserva `pending` de 3 slots (`total = 160`) | Fluxo H-2 completo | `amount`/`duration_hours`/`slots` batem com os 3; transição e protocolo OK; os 3 slots seguem bloqueados. |
| **X-13** (comprovante — modo base64) | `GOOGLE_DRIVE_*` **ausentes** em `pagamentos` | H-2 | Upload cai no fallback base64-no-documento; `prefix` = `number` gerado; resto igual. |
| **X-14** (review idempotente) | Reserva já `approved` + `manual_payment` `pending` (cenário raro) | review `approved` | Não re-transiciona a reserva (já está no alvo); revisa o comprovante. review `rejected` nesse caso → `409 "A reserva nao esta pendente de revisao"`. |

---

## 5. Checklist de regressão (fluxos vizinhos)

| ID | Área | Verificação |
|---|---|---|
| **R-1** | `POST /checkout` (Getnet/mock) | Cria reserva + checkout normalmente (AC-29). |
| **R-2** | `POST /webhooks/getnet` | Processa webhook e atualiza status/metadata da reserva — intacto (não usa refund). |
| **R-3** | `GET /transaction-history`, `GET /transaction-history/:id`, `/proof` | Listagem/leitura/download de comprovante do admin — intactos. |
| **R-4** | `PATCH /reservas/:id` (update genérico do admin) | Editar dados da reserva; `status: 'expired'` → libera slots (allowlist `RELEASING_STATUSES`); `status: 'cancelled'`/`'rejected'` → comportamento anterior. |
| **R-5** | `PATCH /reservas/:id/payment` (metadata, chamada por `pagamentos` no review) | Aceita `payment_method: 'PIX'`; `refund_status: 'manual'` agora é valor válido no DTO. |
| **R-6** | `GET /reservas/protocol/:number` | Reserva com `number` retorna dados + `can_cancel`; `expired`/`cancelled` → `can_cancel: false`. |
| **R-7** | `close-date` (`POST /day/close-date`) | Detecta reservas ativas (incl. `waiting_approve`) nos slots do dia; conflito lista reservas mesmo as **sem** `number` (`ICloseDateReserveConflict.number` opcional). |
| **R-8** | `ranking_agendamento` — `create-bulk`, `trocar-dia`, `delete-bulk`, `attach-proof` | Contratos inalterados; `create-bulk` continua gerando `numero_protocolo`; `attach-proof` continua exigindo `status: 'pending'` → `waiting_approve`. |
| **R-9** | `campeonato_agendamento` + kernel de conflito (`EventConflictService`) | Bloqueio/liberação de quadra por campeonato/ranking segue funcionando; `ranking_agendamento` `expired` não deve mais bloquear. |
| **R-10** | `beach-center-server` | `docker-compose.dev.yml` sobe `agendamentos` sem `PAGAMENTOS_API_URL`/`PAGAMENTOS_INTERNAL_API_KEY`; `pagamentos` sobe normal. |
| **R-11** | `pagamentos` — suíte de testes | `authOrInternalApiKey` / `internalApiKeyMiddleware` ainda existem e passam nos specs, mesmo sem rota consumidora. |
| **R-12** | Mensalista / aula-bloqueio | Sem alteração — smoke test de criação/cancelamento de plano e bloqueio de aula. |

---

## 6. Rastreabilidade AC → cenário

| AC | Cenário(s) | Cobertura automatizada equivalente? |
|---|---|---|
| AC-1 — nasce `pending` sem protocolo | H-1 | ✅ `create-reserve.usecase.spec` / `create-reserve.adapter.spec` |
| AC-2 — protocolo em `→ waiting_approve` | H-2, H-3 | ✅ `update-reserve-and-scheduling-status.usecase.spec` |
| AC-3 — protocolo idempotente | X-1 | ✅ mesma spec ("NAO regenera …") |
| AC-4 — `waiting_approve` bloqueia | H-2, X-7 | ✅ `find-active-reserves` / `has-active-reserve` adapter specs + usecase spec |
| AC-5 — aprovação | H-4 | ✅ `update-reserve-and-scheduling-status.usecase.spec` |
| AC-6 — rejeição libera | H-5 | ✅ mesma spec |
| AC-7 — reserva `pending` 24 h expira | H-8 | ✅ `expire-pending-reserves.usecase.spec` |
| AC-8 — `pending` recente não expira | X-2 | ✅ mesma spec (cutoff) |
| AC-9 — expiração não afeta outros status | X-3 | ✅ `list-pending-reserves-older-than.adapter.spec` (filtro `status: 'pending'`) |
| AC-10 — `expired` terminal | E-4 | ✅ `delete-reserve.usecase.spec` + `find-by-protocol.usecase.spec` |
| AC-11 — `waiting_approve` primeiro | H-6, X-9 | ✅ `list-reserves.usecase.spec` |
| AC-12 — contagem de pendentes | H-6, H-7, X-8 | ✅ `list-reserves.usecase.spec` + `list-reserves.controller.spec` |
| AC-13 — filtros continuam | H-7 | ✅ `list-reserves.usecase.spec` / controller spec |
| AC-14 — cancelar aprovada → `manual` | H-9 | ✅ `delete-reserve.usecase.spec` |
| AC-15 — sem pagamento não seta | H-10, X-10 | ✅ `delete-reserve.usecase.spec` |
| AC-16 — janela 2 h preservada | E-6 | ✅ `delete-reserve.usecase.spec` |
| AC-17 — nenhum caminho de refund | E-7 | ✅ ausência de arquivos + `routes.spec`/`container.spec` (pagamentos), `env.spec` (agendamentos) |
| AC-18 — RA `pending` 24 h expira | H-11 | ✅ `expire-pending-ranking-agendamentos.usecase.spec` |
| AC-19 — RA recente/outros não | X-5, X-3 | ✅ mesma spec + `list-pending-ranking-agendamentos-older-than.adapter.spec` |
| AC-20 — RA `waiting_approve` nunca expira | X-4 | ✅ `list-pending-ranking-agendamentos-older-than.adapter.spec` (filtro) |
| AC-21 — submit → `waiting_approve` antes do upload | H-2 | ✅ `submit-manual-payment.usecase.spec` (ordem + prefix/reserve_number) |
| AC-22 — falha upload → `rejected` | E-2, E-3 | ✅ `submit-manual-payment.usecase.spec` |
| AC-23 — reenvio bloqueado | E-1 | ✅ `submit-manual-payment.usecase.spec` |
| AC-24 — review a partir de `waiting_approve` | H-4, H-12 | ✅ `review-manual-payment.usecase.spec` |
| AC-25 — review rejeitando | H-5 | ✅ `review-manual-payment.usecase.spec` |
| AC-26 — review de reserva legada `pending` | H-13 | ✅ `review-manual-payment.usecase.spec` |
| AC-27 — `tsc` / ESLint limpos | E-8 | ✅ CI local (`/speckit-complete` bloqueia) |
| AC-28 — env sem as 2 vars | E-9, R-10 | ✅ `env.spec.ts` (agendamentos) |
| AC-29 — checkout/webhook intactos | H-14, R-1, R-2 | ✅ suítes `create-checkout` / `process-getnet-webhook` (pagamentos) |

**Cenários sem equivalente automatizado direto** (validar manualmente): H-3 (unicidade global de
protocolo entre coleções, com dados reais), H-8/H-11/X-6/X-8 (efeito da varredura com `createdAt`
real no Mongo + liberação de `scheduling`), E-2/E-3 (falha de Drive/Mongo de verdade), E-9/R-10
(boot dos serviços/containers), X-11 (fuso na fronteira dos 24 h), X-13 (fallback base64 real).

---

## 7. Observações para o QA

- **Rede**: para provar AC-14/AC-17 ("nenhuma chamada a `pagamentos`"), rodar `agendamentos` com
  as env de pagamentos ausentes e/ou um proxy que loga qualquer saída para a porta 5001.
- **`pending_approval_count`**: é a contagem **após** a varredura de expiração e **do resultado
  filtrado** — testar com `?status=` para confirmar.
- **Ordem de criação dos `waiting_approve`**: a ordenação só garante "todos os `waiting_approve`
  antes dos demais"; entre eles a ordem é a do banco (tipicamente `createdAt` asc do `find`).
- **Protocolo tardio**: se o `submit` falhar depois de gerar o `number` mas antes de mudar o
  status (E-14), a reserva fica `pending` **com** `number` — não é bug; o próximo `submit`/review
  reaproveita o mesmo `number` (idempotência, X-1).
