# Plano — 004 Sistema Mensalista + separação de Aula de `events_scheduled`

> Gerado por `/speckit-plan`. Escopo ampliado por decisão do usuário: mensalista **e** aula saem
> de `events_scheduled`. Não implementa nada. Base para `/speckit-implement`, `/speckit-unit-tests`,
> `/speckit-component-tests` e `/speckit-test`.

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Serviço | Situação | Mudança |
|---|---|---|
| `services/beach-center-bff-agendamentos` | produção, hexagonal (task 001) | 3 domínios novos: `mensalista` + `mensalista_plano` (CRUD ADMIN) e `aula_bloqueio` (rota **interna** `x-api-key`); kernel de conflito/bloqueio generalizado para 3 fontes; `events_scheduled` reduzido a `OUTRO`; scripts de migração |
| `services/beach-center-bff-aulas` | produção, hexagonal (task 002) | passa a **chamar `agendamentos` via HTTP interno** (novo port `IAulaBloqueioClient` + adapter axios) para criar/atualizar/cancelar o `aula_bloqueio` em `create/update/delete-aula`; `env.ts` ganha `AGENDAMENTOS_API_URL` + `AGENDAMENTOS_INTERNAL_API_KEY` |

Justificativa da fronteira:
- O **bloqueio recorrente de quadra** (dias, horário, conflito, `scheduling.available`) é regra
  de `agendamentos` — já vive lá como `events_scheduled`.
- O **mensalista** não tem microsserviço dono (é só um cadastro de contato, sem login) → CRUD em
  `agendamentos`.
- A **aula** já tem dono (`beach-center-bff-aulas`, task 002 — `IAula`/`IAluno`/vagas/professor).
  Ele **delega** o bloqueio de quadra a `agendamentos` via HTTP interno (mesmo padrão
  `agendamentos → pagamentos`), realizando a dívida deixada explícita no `context.md` da task 002.
- Nenhum outro repo tocado. `beach-center-app` (front) fora de escopo.

### Stack (idêntica ao restante de `agendamentos`)

Node + TypeScript (`ts-node`/`nodemon`), Express, Mongoose/MongoDB, `mongoose-delete`, `yup`,
`firebase-admin` (verificação de ID token), Jest + ts-jest (`coverageThreshold` global 80%),
ESLint flat config com `no-restricted-imports` (domain/applications → infra bloqueado).

### Como o mecanismo atual funciona (base da task)

`events_scheduled` ("eventos_agendados") é a **tabela genérica de exceções**:
`event_type ∈ {AULA_BEACH_TENIS, MENSALISTA, AULA_VOLEI, OUTRO}`, `day_of_week` (string livre
normalizada), `start_time`/`end_time` (string `"HH:MM"`), `court`, `unit`, `status ∈ {CONFIRMED,
CANCELLED}`, `price?`.

- **`EventConflictService`** (`domain/usecases/shared/event-conflict.service.ts`) — recebe um
  `IFindConfirmedEventsByCourtUnitPort`; decide sobreposição (mesmo dia-da-semana + janela de
  horário) contra os `events_scheduled` **CONFIRMED**. Usado por: `create/update` de
  `events_scheduled` (rejeita com `EventConflictError` 400), `create/update` de `scheduling`
  (novo slot nasce `available=false` se conflitar), `list-day-schedulings` (`getConflicts` →
  `exception_conflicts[]` do painel "Exceções" do front).
- **`EventSchedulingImpactService`** (`.../event-scheduling-impact.service.ts`) — recebe um
  `IEventSchedulingImpactInput` abstrato (`day_of_week, start_time, end_time, court, unit,
  status`). `applyEventToSchedulings` → `available=false` nos `scheduling`s de hoje em diante que
  batem; `releaseEventFromSchedulings` → `available=true` nos que não têm reserva ativa **e** não
  estão mais bloqueados (re-checa `eventConflictService.hasConflict`).

### Decisões tomadas nesta etapa (respostas do usuário + design)

1. **`Mensalista` 1→N `MensalistaPlano`, coleções separadas** (`mensalistas`, `mensalista_planos`).
   `MensalistaPlano` referencia `mensalista_id`. Rota aninhada
   `/mensalistas/:mensalista_id/planos` (padrão `/aulas/:aula_id/alunos` da task 002).

   ```ts
   interface IMensalista {
     id: string;
     nome: string;
     telefone: string;
     vencimento_fatura: Date;   // controle manual, SEM integração com pagamentos (task 002 precedente)
     ativo: boolean;            // recálculo lazy na listagem: hoje > vencimento_fatura → false
   }

   interface IMensalistaPlano {
     id: string;
     mensalista_id: string;
     dias: string[];                       // dias da semana, string livre normalizada (mín. 1)
     start_time: string;                   // "HH:MM"  (resposta #3 do usuário)
     end_time: string;                     // "HH:MM"
     court: string;                        // ObjectId de uma quadra
     unit: string;                         // DERIVADO de court no create (court -> unit)
     modalidade: string;                   // string livre (resposta #2)
     equipamentos_proprios: boolean;       // informativo (sem regra)
     status: 'CONFIRMED' | 'CANCELLED';    // só CANCELLED libera a quadra
     price?: number;                       // valor da mensalidade (opcional, como events_scheduled)
   }
   ```

2. **`AulaBloqueio` — coleção separada em `agendamentos`, gerida por HTTP interno.**
   ```ts
   interface IAulaBloqueio {
     id: string;
     aula_id?: string;                    // id da aula no beach-center-bff-aulas (ausente nos migrados)
     dias: string[];
     start_time: string;                  // "HH:MM"
     end_time: string;                    // "HH:MM"
     court: string;
     unit: string;                        // derivado da quadra no create
     modalidade: string;                  // string livre
     status: 'CONFIRMED' | 'CANCELLED';
     source_event_id?: string;            // idempotência da migração
   }
   ```
   - Rota **interna** `/aula-bloqueios` protegida por `internalApiKeyMiddleware` (`x-api-key` ===
     `AGENDAMENTOS_INTERNAL_API_KEY`), **não** ADMIN/Firebase. `POST` (create), `PATCH /:id`
     (update estrutural), `PATCH /:id/delete` (cancel).
   - **Coleção e usecases próprios** (decisão do usuário: não unificar com `mensalista_plano`
     numa tabela com `tipo`). O código de bloqueio/conflito é compartilhado via o kernel (item 3).
   - `beach-center-bff-aulas` ganha `IAulaBloqueioClient` (port de saída) + adapter axios que
     chama `AGENDAMENTOS_API_URL + '/aula-bloqueios'` com `x-api-key`.

3. **Kernel compartilhado generalizado (mínimo), não duplicado.** `events_scheduled` (CONFIRMED,
   `event_type = 'OUTRO'`), `mensalista_plano` (CONFIRMED) e `aula_bloqueio` (CONFIRMED) são
   "bloqueadores recorrentes" com o mesmo shape efetivo. O `EventConflictService` passa a mesclar
   **três** fontes; cada conflito ganha `source: 'EVENT' | 'MENSALISTA' | 'AULA'`.
   - Novos ports `IFindConfirmedMensalistaPlanosByCourtUnitPort` e
     `IFindConfirmedAulaBloqueiosByCourtUnitPort` (mesmo formato do de eventos, com `source`).
   - `EventConflictService.getConflicts` chama os três ports e concatena (planos com `dias[]`
     geram N candidatos, um por dia).
   - `IExceptionConflictEvent` (retorno de `list-day-schedulings`) **ganha o campo `source`**
     (`'EVENT' | 'MENSALISTA' | 'AULA'`) — mesmo array, aditivo.
   - `EventSchedulingImpactService` **não muda de assinatura** — os usecases de plano e de
     bloqueio chamam `applyEventToSchedulings` / `releaseEventFromSchedulings` com seus dados. O
     `release` re-checa `hasConflict` (que agora enxerga as 3 fontes) → liberação correta.

4. **`unit` derivado da quadra.** No `create` do plano/bloqueio: `court → unit` via
   `IFindActiveCourtByIdPort` + o `unit` do documento da quadra (confirmar no implement). 404 se
   a quadra não existe.

5. **Conflito unificado + inadimplência não libera.**
   - Criar plano/bloqueio CONFIRMED: valida quadra, deriva unit, **rejeita** se
     `EventConflictService` acusar sobreposição com evento/plano/**bloqueio de aula** (mesma
     quadra + qualquer dos `dias` + janela) — `MensalistaPlanoConflictError` /
     `AulaBloqueioConflictError` (400, espelham `EventConflictError`).
   - `ativo=false` (inadimplente) **não** libera a quadra. Só `status='CANCELLED'` (cancelamento
     explícito do plano) → `releaseEventFromSchedulings` por dia.
   - Deletar um `mensalista` (pessoa) → cancela todos os seus planos (libera as quadras).

6. **`events_scheduled` reduzido a `OUTRO`.** O enum `EventsScheduledType` e o schema **mantêm**
   `MENSALISTA`/`AULA_BEACH_TENIS`/`AULA_VOLEI` (dados legados + fonte da migração legíveis), mas
   `create/update` de `events_scheduled` **rejeitam** qualquer `event_type ≠ 'OUTRO'`
   (`InvalidInputError` 400, mensagem apontando para `/mensalistas` ou o serviço de aulas). O
   `list` continua aceitando todos os valores (consulta de legados).

7. **Migração — `src/scripts/`, idempotente via `source_event_id`, não apaga os originais.**
   - `migrate-mensalista-from-events.ts` (`npm run migrate:mensalista`): para cada
     `eventos_agendados` `MENSALISTA` sem `mensalista_plano` correspondente → cria
     `Mensalista { nome: "Mensalista importado — <quadra?> <dia> <hora>", telefone: "000000000",
     vencimento_fatura: fim do mês corrente, ativo: true }` + `MensalistaPlano { mensalista_id,
     dias:[day_of_week], start_time, end_time, court, unit, modalidade:"",
     equipamentos_proprios:false, status:<status do evento>, price, source_event_id }`.
   - `migrate-aula-bloqueio-from-events.ts` (`npm run migrate:aula-bloqueio`): para cada
     `eventos_agendados` `AULA_BEACH_TENIS`/`AULA_VOLEI` sem `aula_bloqueio` correspondente →
     cria `AulaBloqueio { dias:[day_of_week], start_time, end_time, court, unit,
     modalidade:(AULA_BEACH_TENIS→"Beach Tenis" | AULA_VOLEI→"Volei"), status:<status do
     evento>, source_event_id }` — **sem `aula_id`** (não há vínculo com uma aula real).
   - Passo manual/pós-task: zerar os `eventos_agendados` migrados.
   - **Sync das aulas existentes** (pergunta #16) — as aulas já cadastradas no
     `beach-center-bff-aulas` (task 002) **não** têm `aula_bloqueio`. A task 002 registrou
     `quadra` como referência livre; hoje não há aula nenhuma criada em produção pelo serviço
     novo (repo bootstrapado, PR mergeada, sem uso). **Assunção:** não precisa de sync
     retroativo — confirmar; se houver aulas, um `npm run sync:aula-bloqueios` no
     `beach-center-bff-aulas` varre e cria via o client.

8. **Autorização:**
   - `mensalista` / `mensalista_plano` → `authMiddleware` + `requireRole('ADMIN')` (Firebase),
     igual a `events_scheduled`.
   - `aula_bloqueio` → `internalApiKeyMiddleware` (`x-api-key`), rota **interna**, sem Firebase.

9. **Falha na chamada `aulas → agendamentos` (pergunta #14).** `create-aula` do
   `beach-center-bff-aulas`: persiste a aula → chama o client do `aula_bloqueio`. Se o client
   falhar (rede, `409` de conflito) → **rollback** (soft-delete/hard-delete do documento da aula
   recém-criado) e propaga o erro (`409`/`502`). Mesmo padrão do `agendamentos → pagamentos`
   (que faz rollback do Firebase). `update-aula`/`delete-aula` idem.

10. **Conversão de horário (pergunta #13).** `IAula.hora_inicio`/`hora_fim` são `Date` no
    `beach-center-bff-aulas` (task 002). O **client** converte para `"HH:MM"` (fuso
    `America/Sao_Paulo`) antes de enviar; `agendamentos` só valida `"HH:MM"` + `start < end`.

11. **Soft-delete:**
    - `mensalista` (pessoa) → `mongoose-delete` (padrão `court`/`unit`).
    - `mensalista_plano` / `aula_bloqueio` → **sem hard delete**; `PATCH /:id/delete` →
      `status='CANCELLED'` + `releaseEventFromSchedulings` (espelha `PATCH /eventos-agendados/:id/delete`).

12. **Fora de escopo (adiado):** cancelamento de ocorrência pontual (só uma data); integração
    com `pagamentos`; separação de **campeonato** de `events_scheduled`; front-end
    (`beach-center-app`); sync retroativo de aulas (se pergunta #16 = "não há aulas").

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras** | 2 repos afetados: `agendamentos` (dono do bloqueio de quadra) e `beach-center-bff-aulas` (dono de `IAula`, delega o bloqueio) | **CONFORME.** Cada repo mexe só no que é seu: `agendamentos` ganha os modelos de bloqueio recorrente (mensalista/aula) — regra dele; `beach-center-bff-aulas` ganha só um **client HTTP de saída** (port + adapter axios), sem lógica de agendamento. A comunicação inter-serviço usa o padrão já existente (`x-api-key` + `internalApiKeyMiddleware` + `AGENDAMENTOS_INTERNAL_API_KEY`), idêntico a `agendamentos → pagamentos`. Realiza a dívida explícita da task 002. |
| **II — Hexagonal / Ports** | ambos os repos já hexagonais | **CONFORME.** Novos domínios nascem na estrutura (models → ports in/out → usecases → adapters → schemas → dto → controllers → routes). Kernel fica em `domain/usecases/shared/`. O client em `beach-center-bff-aulas` é um `domain/ports/output/aula-bloqueio-client.port.ts` + `infra/adapters/aula-bloqueio/*` (adapter axios) — hexagonal correto. **Exceção justificada:** os scripts em `agendamentos/src/scripts/*` (migração) são ferramenta de operação one-shot — importam schemas direto, fora do fluxo hexagonal (mesmo status de um seed script). |
| **III — Test-First / Qualidade** | `coverageThreshold` global 80% nos dois repos | **CONFORME.** Testes no `/speckit-unit-tests`. Em `agendamentos`, specs do kernel (`event-conflict.service`, `event-scheduling-impact.service`, `list-day-schedulings`, `create/update-scheduling`, `create/update-events-scheduled`) **precisam ser atualizados** (2 ports novos, campo `source`, rejeição de `event_type ≠ OUTRO`). Em `beach-center-bff-aulas`, specs de `create/update/delete-aula` atualizados (mock do client), + spec do adapter axios. Scripts de migração testados. ESLint + `tsc` limpos nos dois. |
| **IV — Infra hot-reload** | N/A | Sem mudança de infra. Novos scripts npm em `agendamentos` (`migrate:mensalista`, `migrate:aula-bloqueio`). `beach-center-bff-aulas` ganha `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY` no `env.ts` — refletir no `docker-compose.dev.yml` (task 003) e no `.env.dev.example`. |

**Resultado: sem violação. A exceção dos scripts de migração ao Princípio II está justificada. Nenhum ERRO de bloqueio.**

## Mapa Arquitetural (Hexagonal)

### `services/beach-center-bff-agendamentos/src/`

```
  domain/models/
    mensalista.model.ts                          # IMensalista + data types
    mensalista-plano.model.ts                    # IMensalistaPlano + data types
    aula-bloqueio.model.ts                        # IAulaBloqueio + data types
    recurring-blocker.model.ts                    # IRecurringBlocker { id, dias/day_of_week, start_time, end_time, court, unit, status, source } — tipo do kernel
    events-scheduled.model.ts         (ALTERAR)   # `EventsScheduledType` inalterado; usado só com 'OUTRO' na escrita

  domain/ports/input/
    mensalista.input-port.ts
    mensalista-plano.input-port.ts
    aula-bloqueio.input-port.ts
  domain/ports/output/
    mensalista-persistence.port.ts                # ICreate/IRead/IUpdate/IDelete/IListMensalistasPort + IRecalculateMensalistasStatusPort
    mensalista-plano-persistence.port.ts          # CRUD por mensalista_id + ISetStatusPort + ICancelByMensalistaPort
    aula-bloqueio-persistence.port.ts             # ICreate/IRead/IUpdate/IListAulaBloqueiosPort + ISetAulaBloqueioStatusPort
    recurring-blocker-shared.port.ts              # IFindConfirmedMensalistaPlanosByCourtUnitPort + IFindConfirmedAulaBloqueiosByCourtUnitPort (kernel)

  domain/usecases/shared/                         (ALTERAR)
    event-conflict.service.ts                     # + 2 ports; getConflicts mescla EVENT + MENSALISTA + AULA; itens com `source`
    event-scheduling-impact.service.ts            # sem mudança de assinatura (reuso direto)

  domain/usecases/mensalista/{create,read,update,delete,list}/*.usecase.ts    # list = recálculo lazy `ativo`
  domain/usecases/mensalista-plano/{create,read,update,list,cancel}/*.usecase.ts
  domain/usecases/mensalista-plano/shared/mensalista-plano-conflict.error.ts  # 400
  domain/usecases/aula-bloqueio/{create,read,update,list,cancel}/*.usecase.ts
  domain/usecases/aula-bloqueio/shared/aula-bloqueio-conflict.error.ts        # 400

  domain/usecases/scheduling/{create,update}/     (ALTERAR — só o teste; código já usa EventConflictService)
  domain/usecases/day/list-day-schedulings/       (ALTERAR)   # mapeia `source` no exception_conflicts
  domain/usecases/events_scheduled/{create,update}/  (ALTERAR)  # rejeita event_type ≠ 'OUTRO'
  domain/ports/input/day.input-port.ts            (ALTERAR)   # IExceptionConflictEvent + `source`

  infra/schemas/
    mensalista.schema.ts                          # Mongoose + mongoose-delete
    mensalista-plano.schema.ts                    # status; source_event_id?; índices mensalista_id/court/unit/status
    aula-bloqueio.schema.ts                        # aula_id?; status; source_event_id?; índices court/unit/status
  infra/adapters/mensalista/{create,read,update,delete,list,recalculate-status}/*.adapter.ts
  infra/adapters/mensalista_plano/{create,read,update,list,set-status,cancel-by-mensalista,find-confirmed-by-court-unit}/*.adapter.ts
  infra/adapters/aula_bloqueio/{create,read,update,list,set-status,find-confirmed-by-court-unit}/*.adapter.ts

  applications/dto/
    mensalista.dto.ts                             # yup create/update
    mensalista-plano.dto.ts                       # yup; start/end "HH:MM"; dias min 1; court ObjectId
    aula-bloqueio.dto.ts                          # yup; idem + aula_id opcional
    events-scheduled.dto.ts          (ALTERAR)    # create/update: event_type oneOf(['OUTRO'])
  applications/controllers/mensalista/**, mensalista_plano/**, aula_bloqueio/**    # thin
  applications/middlewares/internal-api-key.middleware.ts   (JÁ EXISTE — reusar)
  applications/routes/
    mensalista.route.ts                           # /mensalistas          (authMiddleware + requireRole('ADMIN'))
    mensalista-plano.route.ts                     # /mensalistas/:mensalista_id/planos   (idem)
    aula-bloqueio.route.ts                        # /aula-bloqueios       (internalApiKeyMiddleware)
    routes.ts                        (ALTERAR)    # monta os 3 sob /api/v1

  config/
    env.ts                            (JÁ TEM AGENDAMENTOS_INTERNAL_API_KEY)
    container.ts                      (ALTERAR)   # ~20 usecases + adapters; injeta os 2 ports no EventConflictService

  scripts/
    migrate-mensalista-from-events.ts             # one-shot idempotente (exceção Princípio II)
    migrate-aula-bloqueio-from-events.ts          # one-shot idempotente

package.json                          (ALTERAR)  # + "migrate:mensalista", "migrate:aula-bloqueio" (ts-node src/scripts/...)
```

### `services/beach-center-bff-aulas/src/` (só o client de saída + wiring)

```
  config/env.ts                       (ALTERAR)  # + AGENDAMENTOS_API_URL, AGENDAMENTOS_INTERNAL_API_KEY
  domain/ports/output/aula-bloqueio-client.port.ts   # ICreateAulaBloqueioClient / IUpdate / ICancel (dados: dias, start "HH:MM", end "HH:MM", court, modalidade, aula_id)
  domain/usecases/aula/create/create-aula.usecase.ts (ALTERAR)  # após persistir a aula → client.create(...); em falha → rollback do documento
  domain/usecases/aula/update/update-aula.usecase.ts (ALTERAR)  # se dias/quadra/hora/modalidade mudaram → client.update(...)
  domain/usecases/aula/delete/delete-aula.usecase.ts (ALTERAR)  # → client.cancel(aula_id)
  domain/usecases/aula/shared/hora-to-hhmm.ts         # Date -> "HH:MM" (America/Sao_Paulo)
  infra/adapters/aula-bloqueio/{create,update,cancel}/*.adapter.ts   # axios POST/PATCH em AGENDAMENTOS_API_URL/aula-bloqueios com x-api-key
  config/container.ts                 (ALTERAR)  # injeta o client nos 3 usecases de aula
```


## Checklist de Implementação

> Ordem: models → ports → kernel compartilhado → usecases mensalista → usecases plano → schemas
> → adapters → dto → controllers → rotas → container → migração → ajustes nos consumidores do kernel.

### Fase 1 — Domínio `mensalista` (pessoa) — ✅ IMPLEMENTADO

- [x] `domain/models/mensalista.model.ts` — `IMensalista` + `ICreate`/`IUpdate` data types
- [x] `domain/ports/input/mensalista.input-port.ts` — 1 por usecase (5)
- [x] `domain/ports/output/mensalista-persistence.port.ts` — `ICreate/IRead/IUpdate/IDelete/IListMensalistasPort` + `IRecalculateMensalistasStatusPort`
- [x] `domain/usecases/mensalista/{create,read,update,delete}/*.usecase.ts` — `assertValidObjectId`; `NotFoundError` em null; `create` deriva `ativo` de `vencimento_fatura >= hoje`. **Nota:** `delete` faz só o soft-delete da pessoa; o cascade de cancelamento de planos entra junto com a Fase 3 (o port `ICancelPlanosByMensalistaPort` ainda não existe).
- [x] `domain/usecases/mensalista/list/list-mensalistas.usecase.ts` — recálculo lazy antes de listar (2 `updateMany`), padrão `markPastSchedulingsUnavailable`
- [x] `infra/schemas/mensalista.schema.ts` — Mongoose, **soft-delete manual** (`deleted: Boolean` + `{ $ne: true }`) — o repo usa esse padrão em `court`/`unit`, não `mongoose-delete` (dep listada mas sem `@types`/uso)
- [x] `infra/adapters/mensalista/{create,read,update,delete,list,recalculate-status}/*.adapter.ts` — 6 adapters, 1 por ação
- [x] `applications/dto/mensalista.dto.ts` — yup: `createMensalistaDTO` / `updateMensalistaDTO` (parcial)
- [x] `applications/controllers/mensalista/{create,read,update,delete,list}/*.controller.ts` — thin
- [x] `applications/routes/mensalista.route.ts` — `POST/GET /mensalistas`, `GET/PATCH /:id`, `PATCH /:id/delete`; `authMiddleware` + `requireRole('ADMIN')`
- [x] `applications/routes/routes.ts` — monta `/mensalistas`
- [x] `config/container.ts` — 6 adapters + 5 usecases + entradas no composition root
- [x] `container.spec.ts` + `routes.spec.ts` — atualizados (chaves + contagem de rotas); suíte **813/813** verde; `tsc` + ESLint limpos

### Fase 2 — Kernel compartilhado (generalização mínima) — `agendamentos` — ✅ IMPLEMENTADO

- [x] `domain/models/events-scheduled.model.ts` — `+ export type RecurringBlockerSource = 'EVENT' | 'MENSALISTA' | 'AULA'` e `source?: RecurringBlockerSource` em `IEventsScheduled`. **Nota:** o "recurring blocker" reaproveita a forma de `IEventsScheduled` (adapters "explodem" `dias[]` em N registros) em vez de um `IRecurringBlocker` novo — menos superfície, mesma semântica.
- [x] `domain/ports/output/mensalista-plano-shared.port.ts` — `IFindConfirmedMensalistaPlanosByCourtUnitPort` (retorna `IEventsScheduled[]` já explodido por dia, `source: 'MENSALISTA'`). O port de `aula_bloqueio` entra na Fase 3b.
- [x] `domain/usecases/shared/event-conflict.service.ts` — construtor `(findConfirmedEventsPort, findConfirmedMensalistaPlanosPort?)` (2º param **opcional** p/ retrocompat dos specs existentes); `getConflicts` faz `Promise.all([events, planos])`, concatena e marca `source` (`event.source ?? 'EVENT'`); matching (dia-da-semana + janela) **inalterado**
- [x] `domain/ports/input/day.input-port.ts` — `IExceptionConflictEvent` **+ `source?: 'EVENT' | 'MENSALISTA' | 'AULA'`**
- [x] `domain/usecases/day/list-day-schedulings/list-day-schedulings.usecase.ts` — mapeia `source: event.source ?? 'EVENT'` no `exceptions[]`
- [x] `domain/usecases/events_scheduled/{create,update}/*.usecase.ts` — rejeita `event_type === 'MENSALISTA'` (`InvalidInputError`). **Nota:** `AULA_BEACH_TENIS`/`AULA_VOLEI` ainda são aceitos aqui — a rejeição total (só `OUTRO`) entra junto com a Fase 3b/5, quando o `aula_bloqueio` assume a escrita e o fluxo de aulas (task 002) para de gravar `events_scheduled` direto.
- [x] `applications/dto/events-scheduled.dto.ts` — `writableEventTypes = ['AULA_BEACH_TENIS','AULA_VOLEI','OUTRO']` (create/update) vs `listableEventTypes` com os 4 (list). `MENSALISTA` fora do writable.

### Fase 3 — Domínio `mensalista_plano` — ✅ IMPLEMENTADO

- [x] `domain/models/mensalista-plano.model.ts` — `MensalistaPlanoStatus`; `IMensalistaPlano` (`mensalista_id`, `dias: string[]`, `start_time`/`end_time` string, `court`, `unit` derivado, `modalidade`, `equipamentos_proprios`, `status`, `price?`, `source_event_id?`); `ICreateMensalistaPlanoData` (sem `unit`/`status`); `IUpdateMensalistaPlanoData` = só `modalidade?`/`equipamentos_proprios?`/`price?` (campos estruturais **não** editáveis)
- [x] `domain/ports/input/mensalista-plano.input-port.ts` — create/read/update/list/cancel
- [x] `domain/ports/output/mensalista-plano-persistence.port.ts` — `ICreate/IRead/IUpdate/IListMensalistaPlanosPort` (por `mensalista_id`), `ISetMensalistaPlanoStatusPort`, `ICancelMensalistaPlanosByMensalistaPort`, `IFindUnitIdByCourtIdPort`
- [x] `domain/usecases/mensalista-plano/shared/mensalista-plano-conflict.error.ts` — `MensalistaPlanoConflictError extends DomainError` (400)
- [x] `domain/usecases/mensalista-plano/create/create-mensalista-plano.usecase.ts` — valida `mensalista` + quadra; deriva `unit` (`IFindUnitIdByCourtIdPort`); `formatRoundedTime`; loop `dias` → `hasConflict` → `MensalistaPlanoConflictError`; persiste `CONFIRMED`; loop `dias` → `applyEventToSchedulings`
- [x] `domain/usecases/mensalista-plano/{read,list,update}/*.usecase.ts` — thin + `assertValidObjectId`; **decisão:** `update` só campos não-estruturais (modalidade/equipamentos/price). Trocar dias/quadra/horário = cancelar + recriar
- [x] `domain/usecases/mensalista-plano/cancel/cancel-mensalista-plano.usecase.ts` — `setStatus('CANCELLED')`; se era `CONFIRMED`, loop `dias` → `releaseEventFromSchedulings(..., status:'CONFIRMED')`
- [x] `infra/schemas/mensalista-plano.schema.ts` — Mongoose (coll. `mensalista_planos`); `mensalista_id`/`court`/`unit` ObjectId; índices `mensalista_id` e `{court, unit, status}`; `source_event_id?`; soft-delete manual
- [x] `infra/adapters/mensalista_plano/{create,read,update,list,set-status,cancel-by-mensalista,find-confirmed-by-court-unit}/*.adapter.ts` (7) + `infra/adapters/unit/find-id-by-court/find-unit-id-by-court.adapter.ts`. O `find-confirmed-by-court-unit` explode cada plano `CONFIRMED` em N `IEventsScheduled` (`event_type: 'MENSALISTA'`, `source: 'MENSALISTA'`)
- [x] `applications/dto/mensalista-plano.dto.ts` — yup: `dias` min 1; `start_time`/`end_time` `"HH:MM"` regex + `start < end`; `court` ObjectId; `modalidade` string; `equipamentos_proprios` boolean; `price` ≥ 0 opcional; `updateMensalistaPlanoDTO` parcial (modalidade/equipamentos_proprios/price)
- [x] `applications/controllers/mensalista_plano/{create,read,list,update,cancel}/*.controller.ts` — thin, `mensalista_id` de `req.params`
- [x] `applications/routes/mensalista-plano.route.ts` — `Router({ mergeParams: true })`, ADMIN, `PATCH /:id/delete` → `cancelMensalistaPlano`; montada em `routes.ts` antes de `/mensalistas`
- [x] `config/container.ts` — 8 adapters + 5 usecases; `EventConflictService` recebe `findConfirmedMensalistaPlanosByCourtUnitAdapter`; `DeleteMensalistaUsecase` agora `(delete, cancelPlanosByMensalista, eventSchedulingImpactService)`; composition root `+ createMensalistaPlano/readMensalistaPlano/listMensalistaPlanos/updateMensalistaPlano/cancelMensalistaPlano`
- [x] `container.spec.ts` (+5 chaves) + `routes.spec.ts` (`KNOWN_PREFIXES` + prefixo aninhado, contagem 42→47, teste dos endpoints de plano). Suíte **814/814** verde; `tsc --noEmit` + ESLint limpos

### Fase 3b — Domínio `aula_bloqueio` — `agendamentos` — ✅ IMPLEMENTADO

- [x] `domain/models/aula-bloqueio.model.ts` — `AulaBloqueioStatus`; `IAulaBloqueio` (`aula_id?`, `dias`, `start_time`/`end_time` string, `court`, `unit` derivado, `modalidade`, `status`, `source_event_id?`); `ICreateAulaBloqueioData`, `IUpdateAulaBloqueioData` (estrutural completo), `ICancelAulaBloqueioTarget` (`id?`/`aula_id?`), `IListAulaBloqueiosFilter`
- [x] `domain/ports/input/aula-bloqueio.input-port.ts` — create/read/list/update/cancel
- [x] `domain/ports/output/aula-bloqueio-persistence.port.ts` — `ICreate/IRead/IListAulaBloqueiosPort`, `IUpdateAulaBloqueioStructuralPort`, `ISetAulaBloqueioStatusPort` (deriva unit reaproveitando `IFindUnitIdByCourtIdPort` do `mensalista_plano`)
- [x] `domain/ports/output/aula-bloqueio-shared.port.ts` — `IFindConfirmedAulaBloqueiosByCourtUnitPort` (explode `dias[]`, `event_type: 'AULA_BEACH_TENIS'`, `source: 'AULA'`, `excludeBloqueioId?`)
- [x] `domain/usecases/aula-bloqueio/shared/aula-bloqueio-conflict.error.ts` — `AulaBloqueioConflictError extends DomainError` (400)
- [x] `domain/usecases/aula-bloqueio/create/create-aula-bloqueio.usecase.ts` — valida quadra + deriva unit; `formatRoundedTime`; loop `dias` → `hasConflict` → `AulaBloqueioConflictError`; persiste `CONFIRMED`; loop `dias` → `applyEventToSchedulings`
- [x] `domain/usecases/aula-bloqueio/{read,list}/*.usecase.ts` — `read` por id; `list` por `aula_id`/`court`/`status`
- [x] `domain/usecases/aula-bloqueio/update/update-aula-bloqueio.usecase.ts` — **estrutural**: revalida conflito (`excludeBloqueioId`) → persiste → `releaseEventFromSchedulings` do estado antigo → `applyEventToSchedulings` do novo
- [x] `domain/usecases/aula-bloqueio/cancel/cancel-aula-bloqueio.usecase.ts` — `setStatus('CANCELLED')` + `releaseEventFromSchedulings` por dia; alvo por `id` **ou** `aula_id` (cancela todos os CONFIRMED da aula), `InvalidInputError` se nenhum
- [x] `infra/schemas/aula-bloqueio.schema.ts` — Mongoose (coll. `aula_bloqueios`); índices `{court, unit, status}`, `aula_id`, `source_event_id`
- [x] `infra/adapters/aula_bloqueio/{create,read,list,update-structural,set-status,find-confirmed-by-court-unit}/*.adapter.ts` (6)
- [x] `applications/dto/aula-bloqueio.dto.ts` — yup: `dias` min 1; `start_time`/`end_time` `"HH:MM"` + `start < end`; `court` ObjectId; `modalidade` string; `createAulaBloqueioDTO` `+ aula_id` ObjectId opcional; `updateAulaBloqueioDTO` = só os estruturais
- [x] `applications/controllers/aula_bloqueio/{create,read,list,update,cancel}/*.controller.ts` — thin; `cancel` lê `?aula_id=` (senão `:id`); `list` lê `?aula_id=`/`?court=`/`?status=`
- [x] `applications/routes/aula-bloqueio.route.ts` — `Router` + `use(internalApiKeyMiddleware)`; `POST /`, `GET /`, `GET /:id`, `PATCH /:id/delete`, `PATCH /:id`; montada em `routes.ts` como `/aula-bloqueios`
- [x] `config/container.ts` — 6 adapters + 5 usecases; `EventConflictService` recebe 3º arg `findConfirmedAulaBloqueiosByCourtUnitAdapter`; composition root `+ createAulaBloqueio/readAulaBloqueio/listAulaBloqueios/updateAulaBloqueio/cancelAulaBloqueio`
- [x] `domain/usecases/shared/event-conflict.service.ts` — 3ª fonte opcional; `IEventConflictTarget + excludeBloqueioId?`
- [x] `container.spec.ts` (+5 chaves) + `routes.spec.ts` (`KNOWN_PREFIXES` + teste de endpoints, contagem 47→52). Suíte **815/815** verde; `tsc --noEmit` + ESLint limpos

### Fase 4 — Aplicação e integração — `agendamentos` — ✅ IMPLEMENTADO

- [x] `applications/controllers/mensalista/**`, `mensalista_plano/**`, `aula_bloqueio/**` — thin
- [x] `applications/routes/mensalista.route.ts` — `/mensalistas` CRUD; `authMiddleware` + `requireRole('ADMIN')` (Fase 1)
- [x] `applications/routes/mensalista-plano.route.ts` — `Router({ mergeParams: true })`, `/mensalistas/:mensalista_id/planos`; ADMIN (Fase 3)
- [x] `applications/routes/aula-bloqueio.route.ts` — `/aula-bloqueios` `POST`, `GET`, `GET/PATCH /:id`, `PATCH /:id/delete`; **`internalApiKeyMiddleware`** (não Firebase). `?aula_id=` no `PATCH /:id/delete` cancela por aula (Fase 3b)
- [x] `applications/routes/routes.ts` — monta os 3 (`/mensalistas/:mensalista_id/planos`, `/mensalistas`, `/aula-bloqueios`)
- [x] `config/container.ts` — adapters + usecases dos 3 domínios; `EventConflictService` recebe `findConfirmedMensalistaPlanosByCourtUnitAdapter` **e** `findConfirmedAulaBloqueiosByCourtUnitAdapter`; wiring com `EventSchedulingImpactService`
- [x] ESLint + `tsc --noEmit` limpos em `agendamentos`

### Fase 5 — Client em `beach-center-bff-aulas` — ✅ IMPLEMENTADO

- [x] `config/env.ts` — `+ AGENDAMENTOS_API_URL?`, `+ AGENDAMENTOS_INTERNAL_API_KEY?` **opcionais** (mantém `env.spec` verde e o serviço sobe sem a integração); o adapter do client lança `DomainError(500)` claro se faltarem no momento da chamada
- [x] `domain/ports/output/aula-bloqueio-client.port.ts` — `ICreateAulaBloqueioClientPort`, `ICancelAulaBloqueioClientPort` (+ `IAulaBloqueioClientCreatePayload`). **Desvio:** sem `IUpdateAulaBloqueioClientPort` — o update do bloqueio é `cancel(aula_id)` + `create(...)` orquestrado no `update-aula.usecase` (aula → 1 bloqueio; mantém os adapters sem orquestração)
- [x] `domain/usecases/aula/shared/hora-to-hhmm.ts` — `Date → "HH:MM"` via `Intl.DateTimeFormat('en-GB', { timeZone: 'America/Sao_Paulo' })`
- [x] `domain/usecases/aula/create/create-aula.usecase.ts` — após `createAulaPort`: `createAulaBloqueioClient.execute({ aula_id, ... })`; em erro → **rollback hard-delete** (`IHardDeleteAulaPort`, best-effort com log) e propaga
- [x] `domain/usecases/aula/update/update-aula.usecase.ts` — se `dias`/`quadra`/`hora_inicio`/`hora_fim`/`modalidade` mudaram → `cancelAulaBloqueioClient` + `createAulaBloqueioClient` (compara com o `existing`)
- [x] `domain/usecases/aula/delete/delete-aula.usecase.ts` — após soft-delete: `cancelAulaBloqueioClient.execute(aula.id)` — **bloqueante** (propaga erro; log no adapter)
- [x] `infra/adapters/aula-bloqueio/{create,cancel}/*.adapter.ts` — axios `POST /aula-bloqueios` e `PATCH /aula-bloqueios/by-aula/:aula_id/delete` com `x-api-key`; `create` mapeia `409` → `ConflictError` de domínio
- [x] `infra/adapters/aula/hard-delete/hard-delete-aula.adapter.ts` — `AulaModel.findByIdAndDelete` (rollback do create)
- [x] `config/container.ts` — injeta os 2 clients + hard-delete nos 3 usecases de aula
- [x] `package.json` (`aulas`) — `+ axios`
- [x] ESLint + `tsc --noEmit` + Jest (164/164) limpos em `beach-center-bff-aulas` (3 specs de usecase ajustados p/ os novos ports mockados)
- [x] `beach-center-server/docker-compose.dev.yml` + `.env.dev.example` (task 003) — serviço `aulas` ganha `AGENDAMENTOS_API_URL=http://agendamentos:5000/api/v1` + `AGENDAMENTOS_INTERNAL_API_KEY=dev-agendamentos-key`
- [x] **Flip do `events_scheduled`** (`agendamentos`) — `create/update-events-scheduled` rejeitam `event_type !== 'OUTRO'`; `writableEventTypes = ['OUTRO']` no DTO; specs ajustados; suíte 815/815

### Fase 6 — Migração — `agendamentos` — ✅ IMPLEMENTADO (scripts)

- [x] `src/scripts/migrate-mensalista-from-events.ts` — idempotente (`source_event_id` no `MensalistaPlano`), 1 `Mensalista` placeholder (`vencimento_fatura` 2999, `ativo: true`) + 1 `MensalistaPlano` (`dias:[day_of_week]`, herda `status`/`price`) por `eventos_agendados` `MENSALISTA`; **não** apaga o original; log de resumo
- [x] `src/scripts/migrate-aula-bloqueio-from-events.ts` — idempotente (`source_event_id`), 1 `AulaBloqueio` (sem `aula_id`, `modalidade` `Beach Tênis`/`Vôlei` do `event_type`) por `eventos_agendados` `AULA_BEACH_TENIS`/`AULA_VOLEI`; **não** apaga o original; log
- [x] `package.json` (`agendamentos`) — `"migrate:mensalista"`, `"migrate:aula-bloqueio"` (`ts-node src/scripts/...`)
- [ ] `beach-center-documentation/beach-center-bff-agendamentos/` — nova doc `mensalista.md`, `mensalista-plano.md`, `aula-bloqueio.md`; atualizar `evento-agendado.md` (só `OUTRO`) — **no `/speckit-documentation`**, não aqui
- [ ] README/nota — ordem: subir o código → `migrate:mensalista` → `migrate:aula-bloqueio` → validar `scheduling.available` → limpar `eventos_agendados` migrados (passo manual) — **no `/speckit-documentation`/`/speckit-complete`**

### Encerramento

- [x] Nenhum `domain/**` ou `applications/**` importa de `infra/**` (`no-restricted-imports` verde) — **ambos** os repos
- [x] Um adapter por verbo/ação — ambos os repos
- [x] `EventConflictService` / `EventSchedulingImpactService` sem regressão nos specs existentes (specs ajustados só para o novo campo `source`; semântica inalterada)
- [x] `create/update-events-scheduled` rejeitam `event_type !== 'OUTRO'` (flip feito na Fase 5, junto com a integração de aulas)
- [x] `list-day-schedulings` devolve `source` em cada `exception` (`'EVENT' | 'MENSALISTA' | 'AULA'`)
- [x] `EventConflictService` mescla as **3** fontes (`events_scheduled` + `mensalista_planos` + `aula_bloqueios` CONFIRMED)
- [x] `create/update/delete-aula` (task 002) com specs verdes — `agendamentos` 815/815, `beach-center-bff-aulas` 164/164
- [x] **Cobertura ≥ 80%** (`/speckit-unit-tests`) — `agendamentos` **98.85% stmts / 93.52% branch** (1001 testes, 204 suites); `beach-center-bff-aulas` **99.28% stmts / 90.98% branch** (187 testes, 54 suites). Specs gerados para todos os usecases/adapters/controllers/dto novos + scripts de migração (idempotência AC-19/AC-20) + kernel de 3 fontes (AC-17/AC-18)

## Critérios de Aceite (formais)

### Mensalista (pessoa) — CRUD

**AC-1 — Criar mensalista**
- **Given** um `ADMIN` autenticado
- **When** `POST /api/v1/mensalistas` com `nome`, `telefone`, `vencimento_fatura`
- **Then** o mensalista é criado (201) com `ativo` derivado (`vencimento_fatura >= hoje → true`)

**AC-2 — Recálculo lazy de `ativo` na listagem**
- **Given** um mensalista com `vencimento_fatura` no passado e `ativo=true` no banco
- **When** `GET /api/v1/mensalistas`
- **Then** o mensalista retorna com `ativo=false` e o banco é atualizado (padrão `markPastSchedulingsUnavailable`)

**AC-3 — Reativação após atualizar `vencimento_fatura`**
- **Given** um mensalista `ativo=false` cujo `vencimento_fatura` é atualizado para data futura
- **When** `GET /api/v1/mensalistas` em seguida
- **Then** retorna `ativo=true`

**AC-4 — Ler / atualizar / deletar mensalista**
- **Given** um mensalista existente
- **When** `GET`, `PATCH`, `PATCH /:id/delete` são chamados por `ADMIN`
- **Then** cada operação retorna o resultado esperado; `delete` soft-deleta o mensalista **e cancela todos os seus planos**, liberando as quadras (AC-11)

### MensalistaPlano — criação e bloqueio de quadra

**AC-5 — Criar plano válido bloqueia a quadra**
- **Given** um mensalista existente e uma quadra existente, sem conflito de horário
- **When** `POST /api/v1/mensalistas/:id/planos` com `dias:["segunda","quarta"]`, `start_time:"08:00"`, `end_time:"09:00"`, `court`, `modalidade`, `equipamentos_proprios`
- **Then** o plano é criado (201, `status='CONFIRMED'`, `unit` derivado da quadra) e **todos os `scheduling`s de segunda/quarta 08:00–09:00 dessa quadra, de hoje em diante, ficam `available=false`**

**AC-6 — Plano conflitante é rejeitado**
- **Given** já existe (a) um `events_scheduled` CONFIRMED (`OUTRO`), (b) outro `MensalistaPlano` CONFIRMED **ou** (c) um `AulaBloqueio` CONFIRMED na mesma quadra + dia-da-semana + janela de horário sobreposta
- **When** `POST /api/v1/mensalistas/:id/planos` com esse mesmo slot
- **Then** a requisição é rejeitada (400, conflito) e nenhum plano é criado

**AC-7 — Slot criado no futuro já nasce bloqueado**
- **Given** um `MensalistaPlano` CONFIRMED de segunda 08:00–09:00 na quadra X
- **When** um `scheduling` de uma segunda futura 08:00–09:00 na quadra X é criado (via `POST /schedulings` ou geração lazy do `list-day-schedulings`)
- **Then** esse `scheduling` nasce `available=false`

**AC-8 — Inadimplência não libera a quadra**
- **Given** um `MensalistaPlano` CONFIRMED cujo mensalista está `ativo=false` (vencimento vencido)
- **When** `GET /api/v1/schedulings` / `list-day-schedulings` para os dias/horário do plano
- **Then** os `scheduling`s continuam `available=false` (só o cancelamento do plano libera)

**AC-9 — Cancelar o plano libera a quadra**
- **Given** um `MensalistaPlano` CONFIRMED bloqueando `scheduling`s sem reserva ativa
- **When** `PATCH /api/v1/mensalistas/:id/planos/:planoId/delete`
- **Then** o plano fica `status='CANCELLED'` e os `scheduling`s afetados **sem reserva ativa e não bloqueados por outro evento/plano** voltam a `available=true`

**AC-10 — Cancelar não libera slot com reserva ativa nem bloqueado por outro**
- **Given** um `scheduling` bloqueado por dois planos sobrepostos, ou com uma reserva ativa
- **When** um dos planos é cancelado
- **Then** o `scheduling` permanece `available=false`

**AC-11 — Deletar o mensalista cancela os planos**
- **Given** um mensalista com 2 planos CONFIRMED
- **When** `PATCH /api/v1/mensalistas/:id/delete`
- **Then** os 2 planos ficam `CANCELLED` e as quadras são liberadas (regras do AC-9/AC-10)

### AulaBloqueio (rota interna) + integração com `beach-center-bff-aulas`

**AC-12 — Criar aula gera o bloqueio de quadra**
- **Given** um `ADMIN` autenticado em `beach-center-bff-aulas`, um professor válido, uma quadra livre
- **When** `POST /api/v1/aulas` (task 002) com `dias`, `hora_inicio`, `hora_fim`, `quadra`, `modalidade`
- **Then** a aula é criada (201) **e** um `AulaBloqueio` CONFIRMED é criado em `agendamentos` (via HTTP interno `x-api-key`), bloqueando os `scheduling`s dos dias/horário da quadra

**AC-13 — Criar aula em quadra ocupada falha (rollback)**
- **Given** a quadra/dia/horário já tem um `AulaBloqueio`, `MensalistaPlano` ou `events_scheduled` CONFIRMED
- **When** `POST /api/v1/aulas` para esse slot
- **Then** `agendamentos` responde `409`, o `create-aula` faz **rollback** (a aula recém-criada é removida) e retorna erro; nenhum `AulaBloqueio` fica pendurado

**AC-14 — Atualizar/deletar aula reflete no bloqueio**
- **Given** uma aula com `AulaBloqueio` CONFIRMED
- **When** `PUT /api/v1/aulas/:id` muda `dias`/`quadra`/`hora_*`/`modalidade` **ou** `DELETE /api/v1/aulas/:id`
- **Then** o `AulaBloqueio` é atualizado (release do estado antigo + re-apply) **ou** cancelado (`status='CANCELLED'` + release), liberando as quadras conforme AC-9/AC-10

**AC-15 — Rota `/aula-bloqueios` é interna**
- **Given** uma requisição sem `x-api-key` correto
- **When** qualquer método de `/api/v1/aula-bloqueios`
- **Then** `401`/`403` (`internalApiKeyMiddleware`); um `ADMIN` com token Firebase mas sem `x-api-key` **não** acessa

### Integração com o kernel de exceções

**AC-16 — `events_scheduled` só aceita `OUTRO`**
- **Given** um `ADMIN` autenticado
- **When** `POST`/`PATCH /api/v1/eventos-agendados` com `event_type ∈ {MENSALISTA, AULA_BEACH_TENIS, AULA_VOLEI}`
- **Then** a requisição é rejeitada (400) com mensagem apontando para `/mensalistas` ou o serviço de aulas; `event_type='OUTRO'` continua funcionando; `GET /eventos-agendados?event_type=MENSALISTA` (consulta de legados) continua permitido

**AC-17 — `list-day-schedulings` reporta a origem do bloqueio (3 fontes)**
- **Given** um dia com `scheduling`s bloqueados por um `MensalistaPlano`, um `AulaBloqueio` e um `events_scheduled` (`OUTRO`)
- **When** `GET /api/v1/dias/:date/agendamentos` (`list-day-schedulings`)
- **Then** `exception_conflicts[].exceptions[]` inclui os três, cada um com `source ∈ {'MENSALISTA','AULA','EVENT'}`; o shape anterior é preservado + o campo `source`

**AC-18 — Conflito unificado nas 3 direções**
- **Given** um `AulaBloqueio` CONFIRMED de terça 19:00–20:00 na quadra Y
- **When** se tenta criar (a) um `MensalistaPlano` **ou** (b) um `events_scheduled` `OUTRO` no mesmo slot
- **Then** ambos são rejeitados (400, conflito); e o inverso (criar `AulaBloqueio` onde já há plano/evento) também

### Migração

**AC-19 — Migração de MENSALISTA converte 1:1 e é idempotente**
- **Given** N registros `eventos_agendados` `MENSALISTA`
- **When** `npm run migrate:mensalista` roda (uma ou duas vezes)
- **Then** N `Mensalista` placeholder + N `MensalistaPlano` (`source_event_id`); rodar de novo não duplica; os `eventos_agendados` originais intactos; os `scheduling`s continuam `available=false`

**AC-20 — Migração de AULA_* converte 1:1 e é idempotente**
- **Given** N registros `eventos_agendados` `AULA_BEACH_TENIS`/`AULA_VOLEI`
- **When** `npm run migrate:aula-bloqueio` roda (uma ou duas vezes)
- **Then** N `AulaBloqueio` (sem `aula_id`, `modalidade` derivada do `event_type`, `source_event_id`); idempotente; originais intactos; `scheduling`s continuam `available=false`

### Qualidade (Princípio III — cobrado em `/speckit-unit-tests`)

**AC-21 — Isolamento de camadas e specs do kernel (2 repos)**
- **Given** os 2 serviços após a task
- **When** ESLint + Jest com cobertura rodam
- **Then** nenhum import `domain/**`→`infra/**` (exceto `src/scripts/**`); specs de `event-conflict.service`, `event-scheduling-impact.service`, `list-day-schedulings`, `create/update-scheduling`, `create/update-events-scheduled` (agendamentos) e `create/update/delete-aula` (aulas) atualizados e verdes; cobertura global ≥ 80% nos dois

## Riscos e observações

- **Task grande e em 2 repos** — `agendamentos` (3 domínios + kernel + 2 migrações) e
  `beach-center-bff-aulas` (client + wiring). É maior que a 002/003. Se ficar inviável num
  `/speckit-implement`, dividir: (A) mensalista + kernel + migração mensalista; (B) aula_bloqueio
  + client em aulas + migração aula. Mas a decisão do usuário foi "tudo na 004".
- **Kernel compartilhado = maior regressão.** `EventConflictService` é usado por 5 fluxos. A
  mudança (2 ports novos + `source`) precisa preservar **exatamente** a semântica de matching
  atual quando as 2 fontes novas estão vazias. Specs existentes são a rede de segurança.
- **`list-day-schedulings` é o usecase mais complexo do repo** — a adição do `source` deve ser
  puramente aditiva no shape.
- **Acoplamento `aulas → agendamentos`** — hoje `beach-center-bff-aulas` não fala com ninguém.
  Passa a ter um ponto de falha externo no `create/update/delete-aula`. Rollback do documento da
  aula na falha do client (como `agendamentos → pagamentos` faz com o Firebase). Testar o
  caminho de falha.
- **Migração sem nome/telefone real** — mensalistas importados nascem com placeholder
  (`telefone: "000000000"` para não violar `required`); o ADMIN completa. `AulaBloqueio`
  importado nasce **sem `aula_id`** (não há aula real). Documentar.
- **`update` estrutural de plano/bloqueio** (dias/quadra/horário) exige release + re-check +
  re-apply — pode ser mais simples **proibir** (só `price`/`modalidade` editáveis; para o resto,
  cancela e recria). Decidir no `/speckit-implement`.
- **Migração não destrutiva** — os scripts não apagam os `eventos_agendados`; a limpeza final é
  passo manual pós-task.
- **`docker-compose.dev.yml` / `.env.dev.example` (task 003)** — precisam de `AGENDAMENTOS_API_URL`
  + `AGENDAMENTOS_INTERNAL_API_KEY` no serviço `aulas` (hoje não tem).
- **`beach-center-app`** — o painel "Exceções" (que consome `list-day-schedulings` e
  `/eventos-agendados`) vai ver `source` novos e não vai mais criar MENSALISTA/AULA por lá.
  Ajuste do front é **task futura** (fora de escopo).

## Próximo passo

Fases 1–6 **implementadas** (`/speckit-implement`, 4 passadas):

| Repo | Estado |
|---|---|
| `agendamentos` | 3 domínios novos (`mensalista`, `mensalista_plano`, `aula_bloqueio`), kernel de conflito com 3 fontes, `events_scheduled` só `OUTRO`, 2 scripts de migração. `tsc`/ESLint limpos, Jest **815/815**. |
| `beach-center-bff-aulas` | client HTTP → `agendamentos` (`create`/`cancel` bloqueio + rollback), `env.ts`, `hora-to-hhmm`. `tsc`/ESLint limpos, Jest **164/164**. |
| `beach-center-server` | `docker-compose.dev.yml` + `.env.dev.example` — `aulas` recebe `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY`. |

**`/speckit-unit-tests`** — ✅ concluído: `agendamentos` 1001 testes / 93.52% branch; `beach-center-bff-aulas` 187 testes / 90.98% branch.

**`/speckit-component-tests`** — ✅ **N/A** (task 100% backend; `beach-center-app` sem mudanças e fora de escopo; Cypress+Cucumber não instalados no ecossistema — mesmo veredicto das tasks 002/003). Os fluxos E2E ficam para o roteiro exploratório manual do `/speckit-test`.

**`/speckit-validate`** — ✅ (revisão autônoma; 1 ajuste de doc-comment).
**`/speckit-test`** — ✅ `exploratory-tests.md` (67 cenários; 21 "só manual").
**`/speckit-complete`** — ✅ Quality Gate local verde nos 2 serviços; commit + push em `feat/task-004-sistema-mensalista` nos 4 repos:
`agendamentos` `29bbea3` · `aulas` `069b8e3` · `beach-center-server` `a621a04` · `beach-center-ia` `d78e8aa`.
Sem CI configurada nos repos (o gate local é o efetivo).

Próximo: **`/speckit-documentation`** — docs de `mensalista`/`mensalista-plano`/`aula-bloqueio` em `beach-center-documentation/beach-center-bff-agendamentos/` + atualizar `evento-agendado.md` (só `OUTRO`) + nota de ordem da migração.
