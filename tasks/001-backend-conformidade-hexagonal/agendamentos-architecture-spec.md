# Arquitetura Hexagonal — `beach-center-bff-agendamentos` (Fase 3)

> Spec de implementação para `/speckit-implement`. Objetivo: permitir que múltiplos agentes
> implementem partes independentes (por recurso) sem falar entre si, chegando a um resultado
> consistente. Segue a Constituição (`.specify/memory/constitution.md`, Princípio II) e o
> `plan.md` da task 001.
>
> **Desvio deliberado do `plan.md`**: o `plan.md` esboça `domain/ports/{input,output}/<recurso>/<ação>.port.ts`
> (uma pasta por recurso, um arquivo por ação). A implementação real de `usuarios` e
> `pagamentos` (já validada e commitada) usa **um arquivo por recurso, agrupando várias
> interfaces** (ex.: `domain/ports/output/user-persistence.port.ts` contém 9 interfaces).
> Esta spec segue o **precedente real**, não o esboço, para manter os 3 serviços consistentes.
> Mesma lógica para `domain/usecases/shared/*` (não `domain/usecases/<recurso>/shared/*`) quando
> a lógica é usada por mais de um recurso.

## Convenção geral (idêntica a usuarios/pagamentos)

- `src/domain/errors.ts` — `DomainError` (base, `statusCode` público) + `InvalidInputError`
  (400), `NotFoundError` (404), `ConflictError` (409), `UnauthorizedError` (401),
  `ForbiddenError` (403). Copiar literalmente de `beach-center-bff-usuarios/src/domain/errors.ts`.
- `src/domain/usecases/shared/handle-usecase-error.ts` — copiar literalmente de usuarios.
- `src/applications/controllers/shared/handle-http-error.ts` — copiar literalmente de usuarios
  (`instanceof yup.ValidationError` → 400; `instanceof DomainError` → `error.statusCode`; senão
  500). **Todo controller vira**: validar (yup) → `container.<usecase>.execute(...)` → `res.json`;
  erros vão para `catch (error) { return handleHttpError(res, error); }`. Sem `instanceof` extra
  no controller, **exceto** o caso único documentado em AC-CloseDate abaixo.
- `src/domain/usecases/shared/object-id.ts` — copiar `assertValidObjectId(id, message)` de
  usuarios (lança `InvalidInputError`). Substitui todos os `if (!id || id.length !== 24) return
  res.status(400)...` espalhados pelos controllers atuais — a validação de formato de ID migra
  para o usecase.
- Todo usecase implementa uma interface de `domain/ports/input/*`, recebe ports de output no
  construtor, chama `handleUsecaseError` no catch.
- Todo adapter implementa uma interface de `domain/ports/output/*`, só acessa
  `infra/schemas/**` ou clients externos (axios).
- `src/config/container.ts` — composition root único (todas as ~34 factories).
- `src/config/env.ts` — copiar padrão de pagamentos (`required`/`optional` helpers).
- `src/config/firebase.ts` — mover `infra/firebase/firebase-admin.ts` para cá (client lazy, como
  em pagamentos: `import "../../config/firebase"` vira só necessário onde `getAuth()` é usado).
- `src/main.ts` — copiar padrão de usuarios (`loadEnv`/`env`, `cors()`, `express.json()`,
  `app.use('/api/v1', routes)`, `mongoose.connect`, `if (require.main === module)`).
- `index.js` raiz — reduzir a `module.exports = require('./dist/main');` (igual usuarios).

---

## 1. Kernel de domínio compartilhado (implementar PRIMEIRO — todos os recursos dependem disto)

### 1.1 `src/shared/date-time.ts` (utilitário técnico neutro — Princípio II regra 6)

Merge de `infra/adapters/events_scheduled/shared/local-date-time.ts` +
`infra/adapters/events_scheduled/shared/event-time.ts` (são puros, sem I/O, sem regra de
negócio — só timezone/formatação). Copiar as funções **exatamente como estão hoje**:

```ts
export const APP_TIME_ZONE = 'America/Sao_Paulo';
export type TimeBoundary = 'start' | 'end';
export function normalizeDayOfWeek(dayOfWeek: string): string;
export function parseLocalDate(day: string): Date;
export function buildLocalDateTime(day: string, minutes: number): Date;
export function getLocalDateKey(date: Date): string;
export function getLocalDayOfWeek(date: Date): string;
export function getLocalTimeMinutes(date: Date, boundary?: TimeBoundary): number;
export function getTodayStart(): Date;
export function parseTimeToMinutes(time: string): { hour: number; minute: number; minutes: number };
export function getRoundedTimeMinutes(value: string | Date, boundary: TimeBoundary): number;
export function getInclusiveEventEndMinutes(value: string | Date): number;
export function formatRoundedTime(time: string, boundary: TimeBoundary): string;
export function timeRangesOverlap(a0: number, a1: number, b0: number, b1: number): boolean;
export function isPastScheduling(date: Date): boolean; // from scheduling/shared/past-schedulings.ts — pure, uses getTodayStart
```

Every other file that imported from `local-date-time`, `event-time`, or
`past-schedulings.isPastScheduling` now imports from `src/shared/date-time.ts`.

### 1.2 `src/domain/models/` additions

- `day.model.ts`: `export interface IDay { id: string; day: Date; opened: boolean; scheduling_ids: string[]; }`
- `public-reserve-link.model.ts`:
  ```ts
  export type PublicReserveLinkStatus = "active" | "processing" | "used";
  export interface IPublicReserveLink {
    token: string; status: PublicReserveLinkStatus; expires_at: Date | null;
    used_at?: Date | undefined; reserve_id?: string | undefined; created_by?: string | undefined;
  }
  ```
- `scheduling.model.ts`: add `export interface INewScheduling { date: Date; start_time: Date; end_time: Date; court: string; unit: string; available: boolean; }` (string ids — replaces infra's `ISchedulingToCreate` which used `mongoose.Types.ObjectId`; the adapter converts to `ObjectId` right before `insertMany`).
- `events-scheduled.model.ts`: no change to `IEventsScheduled`.

### 1.3 Ports needed by the shared kernel — `src/domain/ports/output/`

`court-persistence.port.ts` and `unit-persistence.port.ts` are defined in §2/§3 (court/unit
sections) — the kernel reuses `IFindActiveCourtByIdPort`/`IFindActiveUnitByIdPort` from there.

`scheduling-shared.port.ts` (new file — cross-resource, used only by shared services, not by
scheduling's own CRUD which uses `scheduling-persistence.port.ts` from §5):
```ts
export interface IFindSchedulingsByCourtUnitFromDatePort {
  execute(court: string, unit: string, fromDate: Date): Promise<IScheduling[]>;
}
export interface ISetSchedulingsAvailabilityPort {
  execute(schedulingIds: string[], available: boolean): Promise<void>;
}
export interface IHasActiveReserveForSchedulingPort {
  execute(schedulingId: string): Promise<boolean>; // status in ['pending','approved']
}
```

`events-scheduled-shared.port.ts` (new file):
```ts
export interface IFindConfirmedEventsByCourtUnitPort {
  // status: 'CONFIRMED'; excludeEventId used only by update-events-scheduled
  execute(params: { court: string; unit: string; excludeEventId?: string }): Promise<IEventsScheduled[]>;
}
```

### 1.4 `src/domain/usecases/shared/event-conflict.service.ts`

Replaces `infra/adapters/events_scheduled/shared/event-conflict.ts` (splits query from decision).
Preserve the exact matching semantics of `getConfirmedEventConflicts` (including the
`getSchedulingTimeMinutes` UTC/local disambiguation logic — copy it verbatim, it's pure).

```ts
export interface IEventConflictTarget {
  date?: Date; localDateKey?: string; day_of_week?: string;
  start_time: Date | string; end_time: Date | string;
  court: string; unit: string; excludeEventId?: string;
}
export class EventConflictService {
  constructor(private readonly findConfirmedEventsPort: IFindConfirmedEventsByCourtUnitPort) {}
  async getConflicts(target: IEventConflictTarget): Promise<IEventsScheduled[]>;
  async hasConflict(target: IEventConflictTarget): Promise<boolean>;
}
```
Internally: fetch events via port (passing court/unit/excludeEventId), then apply the exact
day-of-week + time-window filtering that `getConfirmedEventConflicts` does today (pure, uses
`src/shared/date-time.ts`). `EventConflictError` (see §1.7) is NOT thrown here — callers decide
whether a conflict is fatal (create/update events-scheduled treat it as `EventConflictError`;
day/scheduling logic just wants the boolean/list).

### 1.5 `src/domain/usecases/shared/event-scheduling-impact.service.ts`

Replaces `apply-event-to-schedulings.ts` + `release-event-from-schedulings.ts`.

```ts
export interface IEventSchedulingImpactInput {
  day_of_week: string; start_time: string; end_time: string;
  court: string; unit: string; status: EventsScheduledStatus;
}
export class EventSchedulingImpactService {
  constructor(
    private readonly findSchedulingsPort: IFindSchedulingsByCourtUnitFromDatePort,
    private readonly setAvailabilityPort: ISetSchedulingsAvailabilityPort,
    private readonly hasActiveReservePort: IHasActiveReserveForSchedulingPort,
    private readonly eventConflictService: EventConflictService,
  ) {}
  async applyEventToSchedulings(event: IEventSchedulingImpactInput): Promise<void>;   // -> available=false on overlapping schedulings from today onward
  async releaseEventFromSchedulings(event: IEventSchedulingImpactInput): Promise<void>; // -> available=true on overlapping schedulings w/ no active reserve and no other confirmed-event conflict
}
```
Copy the exact filtering/candidate logic from the two original files (day-of-week normalize +
`timeRangesOverlap` + `!isPastScheduling`). `releaseEventFromSchedulings` still needs the
"no other confirmed event still conflicts" check — call
`eventConflictService.hasConflict({...})` per candidate scheduling, exactly as today.

### 1.6 `src/domain/usecases/shared/scheduling-availability.service.ts`

Replaces `infra/adapters/scheduling/shared/scheduling-availability.ts`.

```ts
export class SchedulingAvailabilityService {
  constructor(
    private readonly readSchedulingPort: IReadSchedulingPort, // from scheduling-persistence.port.ts, §5
    private readonly hasActiveReservePort: IHasActiveReserveForSchedulingPort,
    private readonly eventConflictService: EventConflictService,
    private readonly setAvailabilityPort: ISetSchedulingsAvailabilityPort,
  ) {}
  async canSchedulingBeAvailable(schedulingId: string, options?: { localDateKey?: string }): Promise<boolean>;
  async releaseSchedulingsIfAllowed(schedulingIds: string[], options?: { localDateKey?: string }): Promise<void>;
  async releaseSchedulingsIfAllowedStrict(schedulingIds: string[], options?: { localDateKey?: string }): Promise<void>;
  async blockSchedulings(schedulingIds: string[]): Promise<void>;
  async blockSchedulingsWithConfirmedEvents(schedulingIds: string[]): Promise<void>; // used nowhere today except transitively — keep for parity, flag as possibly dead (see §9)
}
```
Logic copied verbatim from the adapter version, just with ports instead of direct
`SchedulingModel`/`ReserveModel` calls.

### 1.7 `src/domain/errors.ts` additions specific to agendamentos

In addition to the 5 generic classes copied from usuarios, add exactly one domain-specific
subclass (needed because its HTTP response carries extra fields beyond `message`, so it cannot
go through the generic `handleHttpError` path unmodified):

```ts
export class CloseDateReserveConflictError extends ConflictError {
  constructor(public readonly reserves: ICloseDateReserveConflict[]) {
    super('Existem reservas vinculadas aos horarios deste dia. Confirme para cancelar e estornar antes de fechar.');
  }
}
```
(`ICloseDateReserveConflict` = same shape as today's `ICloseDateReserveConflict` interface,
moved to `domain/models/reserva.model.ts` or kept local to the day usecase file.)

**Error-mapping table** (old class → new, preserving message/statusCode exactly):

| Old class (file) | New | statusCode |
|---|---|---|
| `SchedulingReferenceError` (scheduling/validators) | `NotFoundError` (court/unit not found, 404) or `InvalidInputError` (court not in unit, 400) — same messages, decided by call site | 404 / 400 |
| `EventConflictError` | `ConflictError`-shaped but **default statusCode in original is 400**, not 409 — keep as `new DomainError(message, 400)` or a thin `class EventConflictError extends DomainError { constructor(m){super(m,400)} }` kept in `domain/usecases/events_scheduled/shared/` — do NOT rename to `ConflictError` (would silently change status if `error.statusCode || 400` fallback is ever removed) | 400 |
| `DaySchedulingRulesError` | `InvalidInputError` (message preserved) | 400 |
| `CloseDateError` | `InvalidInputError` (400) or `NotFoundError` (404, "Nenhum agendamento encontrado para fechar") — statusCode was already dynamic per throw site, preserve each | 400/404 |
| `CloseDateReserveConflictError` | see above, keep dedicated class | 409 |
| `OpenDateError` | `InvalidInputError` (400) | 400 |
| `ListDaySchedulingsError` | `InvalidInputError` (400) | 400 |
| `CreateDayValidationError` | `InvalidInputError` (400) | 400 |
| `DeleteSchedulingError` | `ConflictError` (409, "possui reserva vinculada") | 409 |
| `CreateReserveValidationError` | mixed 400/409, preserve per throw site (`InvalidInputError`/`ConflictError`) | 400/409 |
| `DeleteReserveValidationError` | mixed 400/404/409/502 — **502 has no generic subclass**; keep `class DeleteReserveValidationError extends DomainError` locally in the reserva usecase file, constructed with explicit statusCode like today | varies |
| `UpdateReserveValidationError` | mixed 400/409 | varies |
| `UpdateReserveAndSchedulingStatusValidationError` | mixed 400/404/409 | varies |
| `ReserveSchedulingValidationError` | mixed 400/409 | varies |
| `PublicReserveLinkError` | mixed 404/409 | varies |

Rule of thumb: when the original error always used the same fixed statusCode, map to the closest
generic `DomainError` subclass. When a single class was thrown with **different** statusCodes at
different call sites (e.g. `DeleteReserveValidationError`, `ReserveSchedulingValidationError`),
**keep it as a small local class extending `DomainError`** (like usuarios never needed this
because its errors were uniform) — do not force it into a generic subclass, to avoid changing
statusCodes. This is the one place agendamentos' error layer is intentionally richer than
usuarios/pagamentos.

### 1.8 One controller-side exception to "always use generic `handleHttpError`"

`applications/controllers/day/close-date/close-date.controller.ts` must special-case
`CloseDateReserveConflictError` **before** falling back to `handleHttpError`, to preserve the
current response shape (`message`, `code: 'CLOSE_DATE_RESERVE_CONFLICT'`, `reserves`):

```ts
catch (error) {
  if (error instanceof CloseDateReserveConflictError) {
    return res.status(error.statusCode).json({
      message: error.message, code: 'CLOSE_DATE_RESERVE_CONFLICT', reserves: error.reserves,
    });
  }
  return handleHttpError(res, error);
}
```
Every other controller in the service uses the plain 3-line pattern (see §0).

---

## 2. Recurso `court`

Simple CRUD, no cross-resource deps.

- **Model**: `domain/models/court.model.ts` — `ICourt` unchanged.
- **Output port** — `domain/ports/output/court-persistence.port.ts`:
  ```ts
  export interface ICreateCourtPort { execute(name: string): Promise<ICourt>; }
  export interface IReadCourtPort { execute(id: string): Promise<ICourt | null>; }
  export interface IUpdateCourtPort { execute(data: ICourt): Promise<ICourt | null>; }
  export interface IDeleteCourtPort { execute(id: string): Promise<ICourt | null>; }
  export interface IListCourtsPort { execute(): Promise<ICourt[]>; } // "not deleted" list, used internally by update/delete for the `list` response field
  export interface IFindActiveCourtByIdPort { execute(id: string): Promise<{ id: string; name: string } | null>; } // used by scheduling-references validator (§1)
  ```
  Note: today's create/read/update/delete adapters each also expose a `get_updated_list`
  helper used only to build the `list` field in the HTTP response. Model this as the single
  `IListCourtsPort` reused by update/delete usecases (constructor takes both the specific port
  and `IListCourtsPort`).
- **Input port** — `domain/ports/input/court.input-port.ts`: `ICreateCourtUseCase`,
  `IReadCourtUseCase`, `IUpdateCourtUseCase`, `IDeleteCourtUseCase`.
- **Usecases** (`domain/usecases/court/{create,read,update,delete}/*.usecase.ts`): thin,
  `assertValidObjectId` for read/update/delete (today's ad-hoc `id.length!==24` controller
  check moves here), `NotFoundError` when adapter returns null. Preserve response shape:
  `update.execute` returns `{ updated: ICourt | null; list: ICourt[] }`,
  `delete.execute` returns `ICourt[] | null`.
- **Adapters** (`infra/adapters/court/{create,read,update,delete}/*.adapter.ts`): same Mongo
  logic as today, `implements` the matching port. `find-active-court-by-id.adapter.ts` (new,
  small) — `CourtModel.findOne({_id, deleted:{$ne:true}})`.
- **DTO**: `applications/dto/court.dto.ts` unchanged (`name: required string`).
- **Controllers** (`applications/controllers/court/{create,read,update,delete}/*.controller.ts`):
  thin, `container.createCourt.execute(...)` etc., `handleHttpError` on catch. Preserve exact
  messages (`'Court created successfully'`, `'Quadra deletada com sucesso'`, etc.) and status
  codes (201/200/404/500) — **do not translate `'Court created successfully'` to Portuguese**,
  it's an existing inconsistency but AC-9 forbids changing response bodies.
- **Route**: `applications/routes/court.route.ts` (renamed from `court.rotes.ts`, AC-8). Same
  paths/methods/middlewares as today.

## 3. Recurso `unit`

Same shape as court, plus:
- `IListUnitsPort` takes `{ name?: string; search?: string }` and does the existing
  `normalizeSearchText` fuzzy match — this filtering IS arguably domain logic (business rule:
  what counts as a match), but it's pure string manipulation with no I/O dependency once the
  full list is fetched. **Decision**: keep the normalize+filter logic in the **usecase**
  (`list-units.usecase.ts`), and make the port just `IListUnitsPort { execute(): Promise<IUnit[]> }`
  (returns all non-deleted units, adapter does only `UnitModel.find()` + deleted filter). The
  usecase applies `params.search` filtering in-memory — this moves real logic from infra to
  domain per Principle II, unlike court which is a pure passthrough.
- `IFindActiveUnitByIdPort { execute(id): Promise<{id:string; courts:string[]} | null> }` — used
  by the scheduling-references validator (§1/§5).
- DTO, controllers, routes: same pattern as court. Route file `unit.route.ts` (renamed from
  `unit.routes.ts`, AC-8).

## 4. Recurso `events_scheduled`

- **Ports output** — `domain/ports/output/events-scheduled-persistence.port.ts`:
  ```ts
  export interface ICreateEventsScheduledPort { execute(data: IEventsScheduled): Promise<IEventsScheduled>; }
  export interface IReadEventsScheduledPort { execute(id: string): Promise<IEventsScheduled | null>; }
  export interface IUpdateEventsScheduledPort { execute(data: IEventsScheduled): Promise<IEventsScheduled | null>; }
  export interface IDeleteEventsScheduledPort { execute(id: string): Promise<IEventsScheduled | null>; } // sets status=CANCELLED, returns the event before mutation for impact-release
  export interface IListEventsScheduledPort { execute(filters: Partial<IEventsScheduled>): Promise<IEventsScheduled[]>; }
  ```
  Plus reuse `IFindConfirmedEventsByCourtUnitPort` from §1.3.
- **Usecases** depend on: matching persistence port, `EventConflictService` (§1.4),
  `EventSchedulingImpactService` (§1.5), and a scheduling-references validator dependency (§5.4 —
  injected as `SchedulingReferencesValidator`).
  - `create-events-scheduled.usecase.ts`: validate refs → round times (`formatRoundedTime`, now
    from `src/shared/date-time.ts`) → check conflict (`EventConflictService.hasConflict`, throw
    `EventConflictError` if true) → persist → `applyEventToSchedulings`.
  - `update-events-scheduled.usecase.ts`: same, with `excludeEventId`; on success call
    `releaseEventFromSchedulings(previousEventState)` then `applyEventToSchedulings(newState)`.
  - `delete-events-scheduled.usecase.ts`: mark CANCELLED, then `releaseEventFromSchedulings`.
  - `list`/`read`: passthrough with mapping (`IEventsScheduled` shape already matches domain
    model, no reformatting needed beyond what the port already returns).
- **Adapters**: same Mongo calls, `implements` the ports; conflict/impact logic **removed**
  from adapters (moved to usecases above).
- **DTO**: `applications/dto/events-scheduled.dto.ts` unchanged (3 exports:
  `createEventsScheduledDTO`, `updateEventsScheduledDTO`, `listEventsScheduledDTO`).
- **Controllers**: thin. Preserve `'Evento agendado criado com sucesso'` etc.
- **Route**: `events_scheduled.route.ts` unchanged name (already `<recurso>.route.ts`).

## 5. Recurso `scheduling`

- **Output port** — `domain/ports/output/scheduling-persistence.port.ts`:
  ```ts
  export interface ICreateSchedulingPort { execute(data: INewScheduling): Promise<IScheduling>; }
  export interface IReadSchedulingPort { execute(id: string): Promise<IScheduling | null>; }
  export interface IUpdateSchedulingPort { execute(data: IScheduling): Promise<IScheduling | null>; }
  export interface IDeleteSchedulingPort { execute(id: string): Promise<IScheduling[] | null>; } // keep list-return shape (existing behavior)
  export interface IListSchedulingsPort { execute(params: IListSchedulingsQueryParams): Promise<IScheduling[]>; }
  export interface IHasReserveForSchedulingPort { execute(schedulingId: string): Promise<boolean>; } // used by delete, distinct from IHasActiveReserveForSchedulingPort (this one has no status filter — "any reserve" blocks delete)
  export interface IRemoveSchedulingFromDaysPort { execute(schedulingId: string): Promise<void>; } // DayModel.updateMany $pull, used by delete
  export interface IListUpdatedSchedulingsPort { execute(): Promise<IScheduling[]>; } // full list, used to build `list` response field on update
  ```
  `IReadSchedulingPort` is the one reused across resources (reserva, day, shared services) —
  keep its signature stable: `execute(id: string): Promise<IScheduling | null>`.
- **Domain service** — `domain/usecases/scheduling/shared/scheduling-window.validator.ts`
  (business rule: operating hours per unit + lunch break). This is the **pure, non-DB** half of
  `day-scheduling-rules.ts`'s `validateSchedulingWindow`/`parseHour`/`resolveOperatingHours`
  (the hardcoded `unitOneId` special-case IS a business rule, keep it here, not in `day`, since
  scheduling's own create/update usecases need it directly without touching `day`):
  ```ts
  export class SchedulingWindowValidator {
    validateSchedulingWindow(data: { date: Date; start_time: Date; end_time: Date; unit: string }): void; // throws InvalidInputError
    parseHour(hour: string): number;
    resolveOperatingHours(unit: string): IOperatingHours;
  }
  ```
  **This class is also used by `day-scheduling-rules.service.ts` (§6)** — instantiate once and
  inject, don't duplicate the operating-hours table.
- **Usecases**: `create-scheduling.usecase.ts` and `update-scheduling.usecase.ts` depend on:
  `ICreateSchedulingPort`/`IUpdateSchedulingPort`, a scheduling-references validator (new shared
  class, see §5.4), `SchedulingWindowValidator`, `EventConflictService`, and `isPastScheduling`
  (from `src/shared/date-time.ts`) to compute the final `available` flag exactly as today
  (`hasException || isPastScheduling(date) ? false : data.available`).
  `delete-scheduling.usecase.ts` depends on `IHasReserveForSchedulingPort` (throw `ConflictError`
  409 if a reserve exists — was `DeleteSchedulingError`), `IDeleteSchedulingPort`,
  `IRemoveSchedulingFromDaysPort`. `list`/`read` are thin passthroughs.
- **5.4 — `domain/usecases/shared/scheduling-references.validator.ts`** (new shared file, used
  by scheduling, events_scheduled, day):
  ```ts
  export class SchedulingReferencesValidator {
    constructor(
      private readonly findActiveCourtPort: IFindActiveCourtByIdPort, // §2
      private readonly findActiveUnitPort: IFindActiveUnitByIdPort,   // §3
    ) {}
    async validate(data: { court: string; unit: string }): Promise<void>;
    // throws NotFoundError('Quadra nao encontrada') / NotFoundError('Unidade nao encontrada')
    // / InvalidInputError('Quadra nao pertence a unidade informada')
  }
  ```
- **Adapters**: `infra/adapters/scheduling/{create,read,update,delete,list}/*.adapter.ts` +
  `infra/adapters/scheduling/find-active-court-by-id` — no, that one lives under `court/`
  (§2). Delete two adapters new: `has-reserve-for-scheduling.adapter.ts`,
  `remove-scheduling-from-days.adapter.ts`, `list-updated-schedulings.adapter.ts` (or fold the
  "updated list" query into the update/delete adapters directly if simpler — either is fine as
  long as it's a single-purpose file per Principle II rule 4; a "get_updated_list" **private
  method inside** `UpdateSchedulingAdapter`/`DeleteSchedulingAdapter` is what usuarios did NOT do
  (usuarios' adapters are single-method); prefer a **separate** `list-updated-schedulings.adapter.ts`
  implementing `IListUpdatedSchedulingsPort`, injected alongside the specific port, to strictly
  follow "um adapter por verbo/ação" (AC-5). Apply this same pattern to court/unit/day/reserva/
  events_scheduled wherever `get_updated_list`/`get_by_id` exist as extra methods today.
- **DTO**: `create-scheduling.dto.ts` (rename export `createSchedulingSchema` → keep as-is,
  already yup), `update-scheduling.dto.ts`. The `combineDateAndTime` helper in the controllers
  is presentation-only (turns a time-of-day string into a full `Date`) — **keep it in the
  controller**, it's HTTP-input shaping, not a business rule, and doesn't touch infra.
- **Controllers/Routes**: thin; `scheduling.route.ts` unchanged name.

## 6. Recurso `day`

Most complex resource — orchestrates scheduling + events_scheduled + reserva.

- **Output port** — `domain/ports/output/day-persistence.port.ts`:
  ```ts
  export interface IFindDayByDatePort { execute(date: Date): Promise<IDay | null>; }
  export interface ICreateDayPort { execute(data: { day: Date; opened: boolean; scheduling_ids: string[] }): Promise<IDay>; }
  export interface IAppendSchedulingIdsToDayPort { execute(dayId: string, schedulingIds: string[]): Promise<void>; }
  export interface IListDaysPort { execute(): Promise<IDay[]>; }
  export interface ISetDayOpenedPort { execute(dayId: string, opened: boolean): Promise<void>; }
  export interface IUpdateDayClosedSchedulingIdsPort { execute(dayId: string, closedSchedulingIds: string[]): Promise<void>; }
  ```
  Plus reuse from scheduling (§5): a bulk "find-or-create schedulings for a set of slots" is
  needed by close-date/create-day/list-day-schedulings. Add to
  `scheduling-persistence.port.ts` (§5):
  ```ts
  export interface IFindSchedulingsByExactSlotsPort {
    execute(slots: Array<{ date: Date; start_time: Date; court: string; unit: string }>): Promise<IScheduling[]>;
  }
  export interface IInsertSchedulingsPort { execute(schedulings: INewScheduling[]): Promise<IScheduling[]>; }
  export interface IFindSchedulingsByFilterPort { execute(filter: { ids: string[]; court?: string; unit?: string }): Promise<IScheduling[]>; }
  ```
  Plus, from reserva (§7): `IFindActiveReservesForSchedulingsPort` and the ability to invoke
  `DeleteReserveUseCase` (input port, not adapter — see below).
- **Domain service** — `domain/usecases/day/shared/day-scheduling-rules.service.ts` (replaces
  `infra/adapters/day/shared/day-scheduling-rules.ts`). Constructor deps: `SchedulingWindowValidator`
  (§5, reused — do not redefine `resolveOperatingHours`/`parseHour` here),
  `IFindConfirmedEventsByCourtUnitPort` (§1.3). Methods, same signatures as today:
  ```ts
  class DaySchedulingRulesService {
    parseDay(day: string): Date;                       // -> src/shared/date-time.parseLocalDate + format validation
    validateNotPastDate(date: Date): void;              // throws InvalidInputError
    async buildSchedulings(data: IDaySchedulingRuleData, validateRequestedHours?: boolean): Promise<INewScheduling[]>;
  }
  ```
  `buildSchedulings` now returns `INewScheduling[]` (string court/unit ids, no
  `mongoose.Types.ObjectId`) — the **adapter** that inserts them converts to `ObjectId`.
  `formatSchedulingData` is no longer needed as a service method — usecases map `IScheduling`
  domain objects directly (adapters already return domain-shaped objects per their ports).
- **Reserva dependency**: `close-date.usecase.ts` needs to cancel active reserves when
  `cancel_reserves: true`. Define in `domain/ports/input/reserva.input-port.ts` (§7):
  ```ts
  export interface IDeleteReserveUseCase {
    execute(id: string, options?: { skipCancellationTimeValidation?: boolean }): Promise<IReserve[] | null>;
  }
  ```
  `CloseDateUsecase` takes `IDeleteReserveUseCase` as a constructor dependency (injected from
  the container with the *real* `DeleteReserveUsecase` instance — domain-to-domain composition
  via interface, no infra import). Also needs
  `IFindActiveReservesForSchedulingsPort` (§7 output port) to detect the conflict in the first
  place.
- **Usecases** (`domain/usecases/day/{close-date,create,list-days,list-day-schedulings,open-date}/*.usecase.ts`):
  - `close-date`: orchestrates `DaySchedulingRulesService.buildSchedulings` (only when
    court+unit scoped), `IFindSchedulingsByExactSlotsPort`/`IInsertSchedulingsPort`,
    `IFindDayByDatePort`/`ICreateDayPort`/`IAppendSchedulingIdsToDayPort`,
    `IFindSchedulingsByFilterPort`, `IFindActiveReservesForSchedulingsPort`,
    `IDeleteReserveUseCase` (loop + call for each active reserve when `cancel_reserves`),
    `IUpdateDayClosedSchedulingIdsPort`, `ISetDayOpenedPort` (only when no court/unit scope, sets
    `opened=false`), `ISetSchedulingsAvailabilityPort` (§1.3, bulk `available:false`). Throws
    `CloseDateReserveConflictError` (§1.7) when reserves exist and `cancel_reserves` isn't true.
    Preserve every validation/order exactly as `CloseDateAdapter.execute` today.
  - `create-day`: orchestrates validation (`SchedulingReferencesValidator` per item,
    `DaySchedulingRulesService.buildSchedulings(item, true)`), duplicate-slot check (pure, keep
    in usecase), find-existing/insert-missing, `SchedulingAvailabilityService.releaseSchedulingsIfAllowed`,
    day upsert. Same result shape `ICreateDayResult[]` (rename to a domain type, e.g.
    `{ day: IDay; schedulings: IScheduling[] }[]`, controller maps back to old JSON keys if
    needed — **or** keep the flattened shape `{ id, day, opened, scheduling_ids, schedulings }[]`
    to avoid changing the response body; flattened is safer for AC-9).
  - `list-days`: `SchedulingAvailabilityService`-adjacent `markPastSchedulingsUnavailable` — this
    bulk "set available=false where date<today" is generic enough to live as
    `ISetSchedulingsAvailabilityPort`-adjacent method; simplest is a dedicated
    `IMarkPastSchedulingsUnavailablePort` in `scheduling-persistence.port.ts` (bulk update,
    infra-only, no decision logic) called at the top of `list-days` and
    `list-day-schedulings` usecases exactly like today.
  - `list-day-schedulings`: the most complex — combines day-scheduling-rules, event-conflict
    (`getAvailableExceptionConflicts` → `EventConflictService.getConflicts` per scheduling),
    `SchedulingAvailabilityService.releaseSchedulingsIfAllowedStrict`/`blockSchedulings`. Keep the
    exact control flow from `ListDaySchedulingsAdapter.execute`, translated to usecase + ports.
  - `open-date`: symmetric to close-date but simpler (no reserve interaction).
- **Adapters**: one per port above, pure Mongo/`DayModel`/`SchedulingModel` I/O, no decision
  logic, no `console.error` swallow-and-rethrow needed (let errors propagate, usecase wraps via
  `handleUsecaseError`).
- **DTO**: unchanged (`close-date.dto.ts`, `create-day.dto.ts`, `list-days.dto.ts`,
  `list-day-schedulings.dto.ts`, `open-date.dto.ts`).
- **Controllers/Routes**: thin except the close-date special case (§1.8). `day.route.ts`
  unchanged name.

## 7. Recurso `reserva`

- **Output port** — `domain/ports/output/reserve-persistence.port.ts`:
  ```ts
  export interface ICreateReservePort { execute(data: ICreateReserveData): Promise<IReserve>; }
  export interface IReadReservePort { execute(id: string): Promise<IReserve | null>; }
  export interface IUpdateReservePort { execute(data: IUpdateReserveData): Promise<IReserve | null>; }
  export interface IDeleteReservePort {
    execute(id: string, refundData?: IRefundOutcome): Promise<IReserve[] | null>; // sets status=cancelled + refund fields
  }
  export interface IListReservesPort { execute(params: IReserveQueryParams): Promise<IReserve[]>; }
  export interface IFindReserveByIdPort { execute(id: string): Promise<IReserve | null>; } // "get_by_id" helper, used by delete/update usecases before mutating
  export interface IFindReserveByProtocolPort { execute(number: string): Promise<IReserve | null>; }
  export interface IFindSchedulingsByIdsPort { execute(ids: string[]): Promise<IScheduling[]>; } // "get_schedulings_by_ids" helper
  export interface IListUpdatedReservesPort { execute(): Promise<IReserve[]>; }
  export interface IFindActiveReservesForSchedulingsPort {
    execute(schedulingIds: string[]): Promise<IReserve[]>; // status in ['pending','approved'] — used by day/close-date (§6)
  }
  ```
  `ICreateReserveData`/`IUpdateReserveData`/`IRefundOutcome`/`IReserveQueryParams` move to
  `domain/models/reserva.model.ts` (currently scattered across `infra/adapters/reserva/types/*`
  and `domain/usecases/reserva/types/query-params.type.ts` — consolidate, they're identical
  shapes duplicated 2-3x today).
- **Output port (external)** — `domain/ports/output/payment-refund.port.ts`:
  ```ts
  export interface IRefundPaymentPort {
    execute(data: { reserve_id: string; payment_id: string; amount: number; payment_method?: 'PIX' | 'CREDIT_CARD' }):
      Promise<{ refund_id?: string; refund_status: 'pending' | 'approved' | 'failed' | 'skipped'; refunded_at?: string }>;
  }
  ```
  Adapter: `infra/adapters/payment/refund-reserve-payment.adapter.ts` — same axios call as
  today's `RefundReservePaymentAdapter`, reading `PAGAMENTOS_API_URL`/`PAGAMENTOS_INTERNAL_API_KEY`
  from `config/env.ts` (not `process.env` directly — centralize per Fase-0 rule).
- **Input port** — `domain/ports/input/reserva.input-port.ts`: `ICreateReserveUseCase`,
  `IReadReserveUseCase`, `IUpdateReserveUseCase`, `IDeleteReserveUseCase` (§6 signature),
  `IListReservesUseCase`, `IFindReserveByProtocolUseCase`.
- **Domain service** — `domain/usecases/shared/reserve-scheduling.validator.ts` (rename from
  `reserva/shared/`, since day's close-date doesn't use it but events_scheduled-adjacent flows
  might in the future; keeping it under `shared/` is consistent with §1's other cross-cutting
  services — if implementers prefer `reserva/shared/` because it's reserva-specific today, that's
  also acceptable, **pick one and be consistent with where `EventConflictService` etc. live**).
  Same 5 methods as today (`validateSchedulingLimit`, `checkIfSchedulingExist`,
  `validateSchedulingsAreNotPast`, `validateSchedulingsDoNotConflictWithExceptions`,
  `validateSchedulingsAreAvailable`, `validateForReserve`), now depending on `IReadSchedulingPort`
  (§5) and `EventConflictService` (§1.4) instead of adapters directly. Errors become
  `InvalidInputError`/`ConflictError` per the mapping table (§1.7) instead of
  `ReserveSchedulingValidationError` — **or** keep `ReserveSchedulingValidationError extends
  DomainError` locally since its statusCode varies (400/409) per validation — implementer's
  choice, but `create-reserve.usecase.ts` currently duplicates this validator's logic inline
  (dead duplication, see §9) — **the duplication must be removed**, `CreateReserveUsecase` must
  call the shared validator, not reimplement it.
- **Usecases**:
  - `create-reserve.usecase.ts`: delete the inline duplicate validation methods, delegate to
    `ReserveSchedulingValidator.validateForReserve`; on `false` return `null` (preserves 404
    "Agendamento não encontrado" in controller); generate protocol via `nanoid` (keep
    `customAlphabet` call, it's a pure library call, fine in usecase); call `ICreateReservePort`.
  - `delete-reserve.usecase.ts`: keep `execute`/`executeByProtocol`, cancellation-time check
    (2h business rule, pure), refund orchestration (`IRefundPaymentPort`), same error mapping.
  - `update-reserve.usecase.ts`: keep scheduling-diff logic (`hasSchedulingChanged`/
    `getAddedSchedulingIds`, pure), delegate validation to the shared validator.
  - `find-by-protocol.usecase.ts`: keep `can_cancel`/`cancellation_deadline` computation (pure,
    2h rule — **duplicated with delete-reserve's own cancellation-time check**; consider
    extracting a single `domain/usecases/shared/reserve-cancellation-window.ts` with
    `getCancellationDeadline(schedulings)`/`isWithinCancellationWindow(...)`, reused by both —
    recommended but not mandatory for AC compliance, flag as cleanup if not done).
  - `list-reserves.usecase.ts`, `read-reserve.usecase.ts`: thin passthrough.
  - **`update-status` (shared reserva+scheduling)**: `domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts`
    (was `domain/usecases/shared/update-status.ts`) — keep exact logic, now via
    `IFindReserveByIdPort`, `ReserveSchedulingValidator`, a new
    `IUpdateReserveStatusPort`/`IUpdateReserveAndSchedulingStatusPort` output port (was
    `infra/adapters/shared/update-status.ts`), `SchedulingAvailabilityService`
    (`releaseSchedulingsIfAllowed`/`blockSchedulings` — **not** direct
    `ISetSchedulingsAvailabilityPort` calls, to stay consistent with how reserva's own
    create/update/delete usecases manage availability).
- **Adapters**: one per port; `create`, `read`, `update`, `delete`, `list`,
  `find-by-id`, `find-by-protocol`, `find-schedulings-by-ids`, `list-updated`,
  `find-active-reserves-for-schedulings` — split today's multi-method adapters
  (`DeleteReserveAdapter` has 4 methods, `UpdateReserveAdapter` has 3) into one-method adapters
  per AC-5, exactly as usuarios did (e.g. usuarios has both `find-user-by-email` and
  `find-user-by-firestore-id` as separate adapter files even though they're both "read user").
- **DTO**: unchanged (`create-reserva.dto.ts`, `update-reserva.dto.ts`,
  `update-payment-metadata.dto.ts`, `find-by-protocol.dto.ts`). `formatEquipment` (currently
  duplicated in 3 controllers: create-reserve, update-reserve, public-reserve-link) is
  presentation shaping of validated yup output — **move it to
  `applications/dto/reserva-equipment.mapper.ts`** (or similar) shared by the 3 controllers, to
  remove the triplication (not a business rule, just DTO→domain mapping, stays in
  `applications/`).
- **`update-payment-metadata.controller.ts`**: today this bypasses the usecase/adapter layers
  entirely (`ReserveModel.findByIdAndUpdate` directly in the controller — a Principle II
  violation on its own, controllers must never touch `infra/schemas`). Must get a proper
  `UpdateReservePaymentMetadataUsecase` + `IUpdateReservePaymentMetadataPort` output port. This
  route is called by `pagamentos` via `internalApiKeyMiddleware` — preserve the exact payload
  shape yup validates today.
- **Controllers/Routes**: thin (with the exception noted). `reserva.route.ts` unchanged name,
  but its handler `update-reserve.ts` → **rename to `update-reserve.controller.ts`** (AC-8),
  update the import in `reserva.route.ts` accordingly. `shared/update-status.ts` controller →
  `applications/controllers/shared/update-reserve-and-scheduling-status.controller.ts` (rename
  for clarity, optional but recommended since "update-status" is ambiguous across 3 resources).

## 8. Recurso `public-reserve-link`

- **Model**: `domain/models/public-reserve-link.model.ts` (§1.2).
- **Output port** — `domain/ports/output/public-reserve-link-persistence.port.ts`:
  ```ts
  export interface ICreatePublicReserveLinkPort { execute(data: { expires_at?: Date | null; created_by?: string }): Promise<IPublicReserveLink>; }
  export interface IFindValidPublicReserveLinkPort { execute(token: string): Promise<IPublicReserveLink | null>; }
  export interface IClaimPublicReserveLinkPort { execute(token: string): Promise<IPublicReserveLink | null>; }
  export interface IMarkPublicReserveLinkUsedPort { execute(token: string, reserveId: string): Promise<void>; }
  export interface IReleasePublicReserveLinkPort { execute(token: string): Promise<void>; }
  export interface IAuthorizePublicReserveLinkPort { execute(token: string, reserveId: string): Promise<boolean>; }
  ```
  (Split today's single `PublicReserveLinkAdapter` — 6 methods — into 6 one-method adapters per
  AC-5, all sharing the same `PublicReserveLinkModel` and a small shared `format()` mapper —
  the mapper itself is fine as a private helper duplicated per adapter, or factored into
  `infra/schemas/public-reserve-link.schema.ts` as a `toDomainPublicReserveLink()` export,
  mirroring usuarios' `toDomainUser()` pattern.)
- **Input port** — `domain/ports/input/public-reserve-link.input-port.ts`.
- **Usecase** (`domain/usecases/public-reserve-link/public-reserve-link.usecase.ts`): same 3
  methods (`createLink`, `validateLink`, `createReserveWithToken`), now depending on the 6 ports
  above plus `ICreateReserveUseCase` (§7 input port — **not** `CreateReserveAdapter`/
  `CreateReserveUsecase` concrete classes as today; this removes the current
  domain→domain-usecase-construction anti-pattern where `PublicReserveLinkUsecase` does
  `new CreateReserveUsecase(this.createReserveAdapter, this.readSchedulingAdapter)` internally —
  instead the container builds `createReserveUsecase` once and injects it here).
- **Controller**: `buildPublicUrl` (reads `PUBLIC_APP_URL`) moves its env read to
  `config/env.ts`. `authorizePublicReserve` currently instantiates
  `new PublicReserveLinkAdapter()` directly in the controller (Principle II violation, applications
  → infra) — must go through `container.authorizePublicReserveLink` usecase instead. `formatEquipment`
  reuse from §7's shared mapper.
- **Route**: `public-reserve-link.route.ts` unchanged name.

---

## 9. Arquivos suspeitos / ambíguos (perguntar antes de decidir sozinho)

- `applications/dto/get-protocol.dto.ts` exports `getByProtocolDTO`/`cancelByProtocolDTO`,
  **byte-for-byte duplicates** of `find-by-protocol.dto.ts`'s `findReserveByProtocolDTO` (same
  shape, different export names) and **is never imported anywhere** (routes/controllers all use
  `find-by-protocol.dto.ts`). Looks like dead code from an earlier refactor. Recommend deleting
  it during migration (not carrying dead files into the new structure) — confirm with the user
  if unsure rather than silently dropping business logic.
- `SchedulingAvailabilityService.blockSchedulingsWithConfirmedEvents` (§1.6, from
  `scheduling-availability.ts`'s `blockSchedulingsWithConfirmedEvents`) has **no call site**
  anywhere in the current codebase (grep confirms only its own definition). Port it over for
  parity but flag it as possibly dead; do not delete silently since it's a public export that
  might be used by code outside this service's visible surface (none found, but low confidence).
- `day.route.ts`'s commented-out block (`// router.get("/calendar/slots", ...)`) is dead code —
  drop it, it references handlers (`getAvailableSlots`, `updateCalendarConfig`) that don't exist
  anywhere in the codebase.
- The `unit` list search feature (`normalizeSearchText`) is real behavior, not dead — do not
  remove it, see §3.
- `refund-reserve-payment.adapter.ts` and `public-reserve-link` are the only two places this
  service talks to another microservice over HTTP (pagamentos, and indirectly — pagamentos calls
  back into agendamentos' `/reservas/:id/payment` and `/links-reserva-publica/:token/reservas/:reserveId/authorization`).
  These cross-service contracts are the highest-risk area for AC-9 regressions — preserve
  request/response shapes exactly, including header names (`x-api-key`) and base URL env vars
  (`PAGAMENTOS_API_URL`, `PAGAMENTOS_INTERNAL_API_KEY`, `AGENDAMENTOS_INTERNAL_API_KEY`).

---

## 10. `config/env.ts` — variáveis de ambiente hoje lidas via `process.env` (grep confirmado)

| Var | Onde hoje | Uso |
|---|---|---|
| `DB` | `index.js` | mongoose connection string |
| `PORT` | `index.js` | HTTP port (fallback 5000) |
| `FIREBASE_PROJECT_ID` | `infra/firebase/firebase-admin.ts` | firebase-admin init |
| `AGENDAMENTOS_INTERNAL_API_KEY` | `applications/middlewares/internal-api-key.middleware.ts` | inbound internal-API auth (pagamentos → agendamentos) |
| `PAGAMENTOS_API_URL` | `infra/adapters/reserva/delete/refund-reserve-payment.adapter.ts` | outbound call to pagamentos `/refunds` (fallback `http://localhost:5001/api/v1`) |
| `PAGAMENTOS_INTERNAL_API_KEY` | same file | outbound `x-api-key` header |
| `PUBLIC_APP_URL` | `applications/controllers/public-reserve-link/public-reserve-link.controller.ts` | builds public link URL (optional, no fallback → `url` field omitted if unset) |

`config/env.ts` should expose all of these plus keep `PORT` defaulting to `5000` (current
default in `index.js`, **not** 5001 like pagamentos — don't copy pagamentos' default verbatim).

## 11. `config/container.ts` — factories necessárias (visão geral, ~34 usecases)

Wiring order matters because of cross-resource deps:

1. **Leaf adapters first**: all persistence adapters for court, unit, scheduling,
   events_scheduled, reserva, day, public-reserve-link, payment-refund, plus
   `find-active-court-by-id`, `find-active-unit-by-id`, `find-confirmed-events-by-court-unit`,
   `find-schedulings-by-court-unit-from-date`, `set-schedulings-availability`,
   `has-active-reserve-for-scheduling`.
2. **Shared domain services** (§1): `EventConflictService`, `EventSchedulingImpactService`,
   `SchedulingAvailabilityService`, `SchedulingWindowValidator`,
   `SchedulingReferencesValidator`, `DaySchedulingRulesService`, `ReserveSchedulingValidator`.
3. **Leaf usecases**: court (4), unit (5), scheduling (5), events_scheduled (5).
4. **Reserva usecases** (depend on scheduling ports + shared validators + payment-refund port):
   create, read, update, delete, list, find-by-protocol.
5. **Day usecases** (depend on scheduling + events_scheduled + reserva's `deleteReserve` input
   port): close-date, create-day, list-days, list-day-schedulings, open-date.
6. **Public-reserve-link usecase** (depends on reserva's `createReserve` input port): create-link
   flows.
7. **Update-status shared usecase** (depends on reserva + scheduling).

`export type Container = typeof container;` at the end, matching usuarios/pagamentos.

## 12. `applications/middlewares/auth.middleware.ts` rewrite

Today it imports `infra/firebase/firebase-admin` and `infra/schemas/user.schema` directly
(Principle II violation, same as pagamentos/usuarios had before their own Fase 1/2). Must become
container-based, exactly like pagamentos' version (already read in this session):
- New usecase `AuthenticateRequestUsecase` (domain) — verify Firebase ID token (port:
  `IVerifyIdTokenPort`, new adapter wrapping `getAuth().verifyIdToken`), then find the app user by
  `id_firestore` (port: `IFindUserByFirestoreIdPort`, new adapter — agendamentos has its own
  `user.schema.ts`, read-only mirror of usuarios' users collection). Throws `UnauthorizedError`/
  `ForbiddenError` exactly matching today's messages (`'Token nao fornecido'`,
  `'Usuario nao encontrado no sistema'`, `'Token invalido ou expirado'`).
- `requireRole(role)` becomes a thin sync check on `req.authUser.user_type`, throwing
  `ForbiddenError('Acesso negado')` — same message as today.
- `internalApiKeyMiddleware`/`authOrInternalApiKey`/`adminOrInternalApiKey` (3 variants used
  across routes, note **agendamentos has 3, pagamentos only has 2** — `adminOrInternalApiKey` is
  agendamentos-specific, used by `reserva.route.ts`'s `GET /:id`) move their `process.env` reads
  to `config/env.ts` (`env.internalApiKey`), keep exact logic, build `adminOrInternalApiKey` on
  top of the new `authMiddleware`/`requireRole('ADMIN')`.
- `AuthenticatedRequest` interface: today exposes `user?: DecodedIdToken` and
  `databaseUser?: IUserDocument`. Renaming to match pagamentos' `authToken`/`authUser` would
  **break nothing externally** (it's a request-augmentation type, not part of any HTTP contract)
  but **would require updating every controller that reads `req.databaseUser?.id_firestore`**
  (only `public-reserve-link.controller.ts`'s `createPublicReserveLink` does this). Either keep
  the old field names or rename+update that one call site — implementer's choice, document
  which was chosen.

---

## 13. Migration mapping (old path → new path)

All paths relative to `services/beach-center-bff-agendamentos/src/`.

### Config / bootstrap
| Old | New |
|---|---|
| `infra/firebase/firebase-admin.ts` | `config/firebase.ts` |
| (new) | `config/env.ts` |
| (new) | `config/container.ts` |
| (new) | `main.ts` |
| `../index.js` (root) | reduced to `require('./dist/main')` |

### Shared kernel
| Old | New |
|---|---|
| `infra/adapters/events_scheduled/shared/local-date-time.ts` | `shared/date-time.ts` (merged) |
| `infra/adapters/events_scheduled/shared/event-time.ts` | `shared/date-time.ts` (merged) |
| `infra/adapters/scheduling/shared/past-schedulings.ts` (`isPastScheduling`) | `shared/date-time.ts` |
| `infra/adapters/scheduling/shared/past-schedulings.ts` (`markPastSchedulingsUnavailable`) | `infra/adapters/scheduling/mark-past-unavailable/mark-past-schedulings-unavailable.adapter.ts` implementing new `IMarkPastSchedulingsUnavailablePort` |
| `infra/adapters/events_scheduled/shared/event-conflict.ts` | split: query → `infra/adapters/events_scheduled/find-confirmed-by-court-unit/find-confirmed-events-by-court-unit.adapter.ts`; decision → `domain/usecases/shared/event-conflict.service.ts` |
| `infra/adapters/events_scheduled/shared/apply-event-to-schedulings.ts` | `domain/usecases/shared/event-scheduling-impact.service.ts` (`applyEventToSchedulings`) |
| `infra/adapters/events_scheduled/shared/release-event-from-schedulings.ts` | `domain/usecases/shared/event-scheduling-impact.service.ts` (`releaseEventFromSchedulings`) |
| `infra/adapters/scheduling/shared/scheduling-availability.ts` | `domain/usecases/shared/scheduling-availability.service.ts` (decision) + small adapters for the raw Mongo ops it used |
| `infra/adapters/scheduling/validators/scheduling-references.validator.ts` | `domain/usecases/shared/scheduling-references.validator.ts` (decision) + `find-active-court-by-id`/`find-active-unit-by-id` adapters |
| `infra/adapters/day/shared/day-scheduling-rules.ts` | split: pure window/hours → `domain/usecases/scheduling/shared/scheduling-window.validator.ts`; slot-building orchestration → `domain/usecases/day/shared/day-scheduling-rules.service.ts` |
| `domain/usecases/shared/update-status.ts` | `domain/usecases/shared/update-reserve-and-scheduling-status.usecase.ts` |
| `infra/adapters/shared/update-status.ts` | `infra/adapters/reserva/update-status/update-reserve-and-scheduling-status.adapter.ts` (implements new port) |
| `applications/controllers/shared/update-status.ts` | `applications/controllers/shared/update-reserve-and-scheduling-status.controller.ts` |
| `applications/dto/update-status.dto.ts` | unchanged path |

### `court`
| Old | New |
|---|---|
| `infra/adapters/court/{create,read,update,delete}/*.adapter.ts` | same path, `implements` new ports |
| (new) | `infra/adapters/court/find-active-by-id/find-active-court-by-id.adapter.ts` |
| `domain/usecases/court/**` | same path, depend on ports |
| `applications/controllers/court/**`, `applications/dto/court.dto.ts` | unchanged paths, thinned |
| `applications/routes/court.rotes.ts` | `applications/routes/court.route.ts` |

### `unit`
| Old | New |
|---|---|
| `infra/adapters/unit/{create,read,update,delete,list}/*.adapter.ts` | same path, `implements` new ports; `list-units.adapter.ts` loses the search-filter logic (moves to usecase) |
| `infra/adapters/unit/types/*.ts` | folded into `domain/models/unit.model.ts` / `domain/ports/output/unit-persistence.port.ts` params |
| (new) | `infra/adapters/unit/find-active-by-id/find-active-unit-by-id.adapter.ts` |
| `domain/usecases/unit/**` | same path; `list-units.usecase.ts` gains the `normalizeSearchText` filtering |
| `applications/controllers/unit/**`, `applications/dto/unit.dto.ts` | unchanged paths |
| `applications/routes/unit.routes.ts` | `applications/routes/unit.route.ts` |

### `events_scheduled`
| Old | New |
|---|---|
| `infra/adapters/events_scheduled/{create,read,update,delete,list}/*.adapter.ts` | same path, `implements` new ports, conflict/impact logic removed |
| `infra/adapters/events_scheduled/types/*.ts` | folded into `domain/models/events-scheduled.model.ts` / port params |
| `domain/usecases/events_scheduled/**` | same path, gain conflict/impact orchestration |
| `applications/controllers/events_scheduled/**`, `applications/dto/events-scheduled.dto.ts` | unchanged |
| `applications/routes/events_scheduled.route.ts` | unchanged |

### `scheduling`
| Old | New |
|---|---|
| `infra/adapters/scheduling/{create,read,update,delete,list}/*.adapter.ts` | same path, `implements` new ports, decision logic removed |
| `infra/adapters/scheduling/types/*.ts` | folded into `domain/models/scheduling.model.ts` (`INewScheduling`) / port params |
| (new) | `infra/adapters/scheduling/has-reserve/has-reserve-for-scheduling.adapter.ts`, `.../remove-from-days/remove-scheduling-from-days.adapter.ts`, `.../list-updated/list-updated-schedulings.adapter.ts`, `.../find-by-exact-slots/*.ts`, `.../insert-many/*.ts`, `.../find-by-filter/*.ts` |
| `domain/usecases/scheduling/**` | same path, depend on ports + `SchedulingWindowValidator` + `SchedulingReferencesValidator` |
| `applications/controllers/scheduling/**`, `applications/dto/{create,update}-scheduling.dto.ts` | unchanged |
| `applications/routes/scheduling.route.ts` | unchanged (drop the dead commented-out block) |

### `day`
| Old | New |
|---|---|
| `infra/adapters/day/close-date/close-date.adapter.ts` | split into `domain/usecases/day/close-date/close-date.usecase.ts` (orchestration) + several small adapters (day/scheduling find-or-create, day upsert, closed-ids update) |
| `infra/adapters/day/create-day/create-day.adapter.ts` | split similarly into `create-day.usecase.ts` + adapters |
| `infra/adapters/day/list-days/list-days.adapter.ts` | `list-days.usecase.ts` + `list-days.adapter.ts` (thin: `DayModel.find().sort()`) |
| `infra/adapters/day/list-day-schedulings/list-day-schedulings.adapter.ts` | `list-day-schedulings.usecase.ts` + adapters |
| `infra/adapters/day/open-date/open-date.adapter.ts` | `open-date.usecase.ts` + adapters |
| `domain/usecases/day/**` | absorb the orchestration logic described in §6 |
| `applications/controllers/day/**` | thinned; `close-date.controller.ts` keeps the one special-case catch (§1.8) |
| `applications/dto/{close-date,create-day,list-days,list-day-schedulings,open-date}.dto.ts` | unchanged |
| `applications/routes/day.route.ts` | unchanged |

### `reserva`
| Old | New |
|---|---|
| `infra/adapters/reserva/create/create-reserve.adapter.ts` | same path, `implements` `ICreateReservePort` |
| `infra/adapters/reserva/delete/delete-reserve.adapter.ts` (4 methods) | split: `delete-reserve.adapter.ts` (execute), `find-by-id/find-reserve-by-id.adapter.ts`, `find-by-protocol/find-reserve-by-protocol.adapter.ts` (shared with §"find-by-protocol" adapter — dedupe, both existed separately with identical query), `find-schedulings-by-ids/find-schedulings-by-ids.adapter.ts` |
| `infra/adapters/reserva/delete/refund-reserve-payment.adapter.ts` | `infra/adapters/payment/refund-reserve-payment.adapter.ts` (moved out of `reserva/`, it's a payment-gateway concern) |
| `infra/adapters/reserva/find-by-protocol/find-by-protocol.adapter.ts` | merge into the deduped `find-by-protocol` adapter above |
| `infra/adapters/reserva/list/list-reserve.adapter.ts` | same path (rename file to `list-reserves.adapter.ts` for consistency, optional) |
| `infra/adapters/reserva/read/read-reserve.adapter.ts` | unchanged |
| `infra/adapters/reserva/update/update-reserve.adapter.ts` (3 methods) | split: `update-reserve.adapter.ts` (execute) + reuse `find-by-id` adapter + `list-updated/list-updated-reserves.adapter.ts` |
| `infra/adapters/reserva/types/*.ts` | folded into `domain/models/reserva.model.ts` |
| `domain/usecases/reserva/shared/reserve-scheduling.validator.ts` | `domain/usecases/shared/reserve-scheduling.validator.ts` (or keep local — see §7 note) |
| `domain/usecases/reserva/types/query-params.type.ts` | folded into `domain/models/reserva.model.ts` |
| `domain/usecases/reserva/**` | dedupe validation logic into the shared validator |
| `applications/controllers/reserva/update/update-reserve.ts` | `applications/controllers/reserva/update/update-reserve.controller.ts` |
| `applications/controllers/reserva/list/list-query-params.types.ts` | delete (duplicate of `domain/models/reserva.model.ts`'s `IReserveQueryParams`) |
| `applications/controllers/reserva/payment-metadata/update-payment-metadata.controller.ts` | thinned, goes through new usecase/port instead of `ReserveModel` directly |
| other `applications/controllers/reserva/**` | thinned |
| `applications/dto/get-protocol.dto.ts` | delete (dead code, see §9) |
| other `applications/dto/*reserva*` | unchanged |
| `applications/routes/reserva.route.ts` | unchanged, update the renamed controller import |

### `public-reserve-link`
| Old | New |
|---|---|
| `infra/adapters/public-reserve-link/public-reserve-link.adapter.ts` (6 methods) | split into 6 one-method adapters under `infra/adapters/public-reserve-link/{create,find-valid,claim,mark-used,release,authorize}/*.adapter.ts` |
| `domain/usecases/public-reserve-link/public-reserve-link.usecase.ts` | same path, depends on ports + `ICreateReserveUseCase` (not concrete classes) |
| `applications/controllers/public-reserve-link/public-reserve-link.controller.ts` | thinned, `authorizePublicReserve` goes through container |
| `applications/dto/public-reserve-link.dto.ts` | unchanged |
| `applications/routes/public-reserve-link.route.ts` | unchanged |

### Schemas (infra, largely untouched — only mapper additions)
| Old | New |
|---|---|
| `infra/schemas/{court,dia,events-scheduled,public-reserve-link,reserva,scheduling,unit,user}.schema.ts` | unchanged paths; consider adding `toDomain*()` mapper exports like usuarios' `user.schema.ts` did, to avoid repeating `{id: String(doc._id), ...}` mapping in every adapter (optional cleanup, not required by any AC) |

### Middlewares
| Old | New |
|---|---|
| `applications/middlewares/auth.middleware.ts` | rewritten per §12, container-based |
| `applications/middlewares/internal-api-key.middleware.ts` | env reads centralized, logic unchanged |

---

## 14. Ordem de implementação sugerida (para paralelizar sem conflito)

1. **Kernel** (§1) + `config/env.ts` + `domain/errors.ts` — bloqueante para tudo, fazer primeiro
   e sozinho.
2. Em paralelo, depois do kernel: **court** (§2) e **unit** (§3) — sem dependências cruzadas
   entre si nem com o resto.
3. Em paralelo: **scheduling** (§5, precisa do kernel + court/unit's `find-active-by-id` ports)
   e **events_scheduled** (§4, precisa do kernel).
4. **reserva** (§7, precisa de scheduling + kernel).
5. **day** (§6, precisa de scheduling + events_scheduled + reserva) e
   **public-reserve-link** (§8, precisa de reserva) — podem ir em paralelo entre si.
6. **`config/container.ts`** completo (§11), **`main.ts`**, **middlewares** (§12), **routes**
   wiring final — feito por último, depois que todas as usecases existem, por quem for "fechar"
   a integração.

Cada etapa acima é independente o suficiente para um agente separado, **desde que os ports do
kernel (§1.3) e os `IFindActiveCourtByIdPort`/`IFindActiveUnitByIdPort`/`IReadSchedulingPort`
(§2/§3/§5) sejam fixados primeiro** — são a interface de acoplamento entre todos os grupos.
