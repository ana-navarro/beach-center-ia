# Task 004 — Sistema Mensalista + separação de Aula de `events_scheduled`

> Gerado por `/speckit-task`. Escopo ampliado por decisão do usuário: além do mensalista, a task
> também tira **aula** da tabela `events_scheduled`. Não implementa nada. Base para `/speckit-plan`.

## Título

**Eliminar (quase) a tabela genérica de exceções (`events_scheduled` / "eventos_agendados")**,
separando-a por domínio dentro de `beach-center-bff-agendamentos`:

1. **Mensalista** — coleção própria `mensalista` (pessoa) + `mensalista_plano` (reserva
   recorrente), CRUD por ADMIN em `agendamentos`.
2. **Aula** — coleção própria `aula_bloqueio` (bloqueio recorrente de quadra), **gerida pelo
   microsserviço `beach-center-bff-aulas` via HTTP interno** (`x-api-key`) — realiza o que a
   task 002 deixou como dívida ("ao criar uma aula, o serviço de aulas deve chamar o
   `agendamentos` para criar o(s) evento(s)").
3. **Migração** dos registros `events_scheduled` de `event_type ∈ {MENSALISTA, AULA_BEACH_TENIS,
   AULA_VOLEI}` para os novos modelos.
4. Ao final, `events_scheduled` fica **só com `OUTRO`** (exceção genérica pontual — o último
   resquício).

## Descrição (a partir do input do usuário)

- **Mensalista** = pessoa que aluga uma ou mais faixas de horário recorrentes numa quadra.
- O agendamento mensal **fica dentro de `agendamentos`** (não é microsserviço novo).
- **Objetivo maior:** "tirar cada vez mais as exceções" — o usuário está **desmontando a tabela
  `events_scheduled`** e separando-a em modelos por domínio (mensalista, campeonato, aula) dentro
  de `agendamentos`. Esta task faz **o pedaço do mensalista**. (A aula, hoje em `events_scheduled`
  como `AULA_BEACH_TENIS`/`AULA_VOLEI`, recebe o mesmo tratamento numa task futura — "a aula vai
  tratar do mesmo jeito que mensalista".)

### Interfaces fornecidas pelo usuário (ponto de partida, não fechado)

```ts
interface IMensalista {
    nome: string;
    telefone: string;
    vencimento_fatura: string;
    ativo: boolean;
}

interface IMensalistaPlano {
    dia: string[];              // dias da semana [segunda..domingo]
    horaInicio: Date;
    horaFim: Date;
    quadra: string;
    modalidade: string;
    equipamentos_proprios: boolean;
}
```

## Serviço(s) alvo (Princípio I)

| Serviço | Papel |
|---|---|
| **`services/beach-center-bff-agendamentos`** | dono das coleções `mensalista`, `mensalista_plano` e `aula_bloqueio`; do kernel de conflito/bloqueio; da migração; da redução do `events_scheduled` a `OUTRO` |
| **`services/beach-center-bff-aulas`** | passa a **chamar `agendamentos` via HTTP interno** (`x-api-key` + `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY`, mesmo padrão de `agendamentos → pagamentos`) para criar/atualizar/cancelar o `aula_bloqueio` quando uma aula é criada/alterada/removida |

Justificativa: o **bloqueio recorrente de quadra** (dias, horário, quadra, conflito) é regra de
`agendamentos` — já vive lá como `events_scheduled`. O **mensalista** não tem microsserviço dono
(a interface é só um cadastro de contato, sem login), então seu CRUD fica em `agendamentos`. A
**aula** já tem dono (`beach-center-bff-aulas`, task 002); ele mantém `IAula`/`IAluno`/vagas e
delega o bloqueio de quadra a `agendamentos`.

## Decisões (respondidas pelo usuário — base para `/speckit-plan`)

1. **`Mensalista` 1 → N `MensalistaPlano`, coleções separadas.** `MensalistaPlano` é documento
   próprio referenciando `mensalista_id`. Rota aninhada `/mensalistas/:mensalista_id/planos`
   (mesmo padrão de `/aulas/:aula_id/alunos` da task 002).
   ```ts
   interface IMensalista {
     id: string;
     nome: string;
     telefone: string;
     vencimento_fatura: Date | string;  // controle manual, sem integração com pagamentos (a confirmar)
     ativo: boolean;                     // recálculo lazy na listagem (padrão markPastSchedulingsUnavailable / task 002)
   }
   interface IMensalistaPlano {
     id: string;
     mensalista_id: string;
     dias: string[];                    // dias da semana
     hora_inicio: Date;
     hora_fim: Date;
     quadra: string;                    // referencia court em agendamentos
     unit: string;                      // DERIVADO da quadra (court -> unit) no momento da criação
     modalidade: string;
     equipamentos_proprios: boolean;
     status: 'CONFIRMADO' | 'CANCELADO'; // (nome a fechar) — só CANCELADO libera a quadra
   }
   ```

2. **Coleção própria + lógica de bloqueio própria + migração.**
   - `mensalista` e `mensalista_plano` são coleções dedicadas em `agendamentos`.
   - O bloqueio de quadra (`scheduling.available = false`) para os slots do plano é feito por
     lógica própria — **adaptando/reusando `EventSchedulingImpactService`** (hoje só serve
     `events_scheduled`).
   - **`event_type: 'MENSALISTA'` deixa de ser criado** via `events_scheduled`. (Enum: manter
     como deprecated para dados legados **ou** remover — a confirmar no plan; o usuário quer
     "tirar a tabela", então a direção é remover ao final da separação completa.)
   - **Migração:** os `events_scheduled` com `event_type = 'MENSALISTA'` existentes em produção
     viram registros de `mensalista_plano` (+ um `mensalista` placeholder — ver pergunta aberta #1).

3. **`unit` derivado da quadra + conflito unificado.**
   - `MensalistaPlano` recebe só `quadra`; o `unit` é resolvido via `court -> unit`
     (`unit/find-active-by-id` a partir do `court`).
   - O `EventConflictService` (hoje consulta apenas `events_scheduled` confirmados) passa a
     considerar **também** os `mensalista_plano` ativos (status ≠ CANCELADO): criar um plano
     conflita com aula/evento/**outro plano de mensalista** sobreposto (mesma quadra +
     dia-da-semana + janela de horário), e vice-versa (`scheduling.create/update` já chamam
     `eventConflictService.hasConflict`).

4. **Inadimplência (`ativo = false`) NÃO libera a quadra.**
   - Igual ao aluno da task 002: `ativo` é só o indicador de pagamento (recálculo lazy a partir
     de `vencimento_fatura`).
   - O que libera a quadra é o **campo de cancelamento do plano** (`status = 'CANCELADO'` /
     `cancelado = true`) — cancelamento explícito. Enquanto o plano está confirmado, os
     `scheduling`s dele ficam `available = false` mesmo com o mensalista inadimplente.
   - Cancelar o plano → `releaseEventFromSchedulings` equivalente (libera os slots sem reserva
     ativa e não bloqueados por outro evento/plano).

### Decisões da parte de AULA (adicionada por decisão do usuário)

A. **`agendamentos` ganha a coleção `aula_bloqueio`**, com o mesmo shape efetivo do
   `mensalista_plano` (dias, horário `"HH:MM"`, quadra, unit derivado, modalidade, status), mas
   **coleção separada** (não uma tabela única com `tipo`) — cada domínio com seus usecases.
   ```ts
   interface IAulaBloqueio {
     id: string;
     aula_id?: string;                   // id da aula no beach-center-bff-aulas (ausente nos registros migrados)
     dias: string[];
     start_time: string;                 // "HH:MM"
     end_time: string;                   // "HH:MM"
     court: string;
     unit: string;                       // derivado da quadra no create
     modalidade: string;                 // string livre (ex.: "Beach Tenis", "Volei")
     status: 'CONFIRMED' | 'CANCELLED';
   }
   ```

B. **Gerido pelo `beach-center-bff-aulas` via HTTP interno.** Endpoints de `aula_bloqueio` em
   `agendamentos` são **internos** (`internalApiKeyMiddleware` — `x-api-key`), não ADMIN/Firebase.
   `beach-center-bff-aulas` ganha um port `IAulaBloqueioClient` + adapter axios
   (`AGENDAMENTOS_API_URL` + `AGENDAMENTOS_INTERNAL_API_KEY`).

C. **Wiring no `beach-center-bff-aulas`:**
   - `create-aula` → após persistir a aula, chama `agendamentos` para criar o `aula_bloqueio`
     (`dias`, `hora_inicio`/`hora_fim` da aula convertidos para `"HH:MM"`, `quadra`, `modalidade`).
     Se `agendamentos` retornar conflito → a criação da aula falha (rollback do documento da aula).
   - `update-aula` que altera `dias`/`quadra`/`hora_*`/`modalidade` → atualiza o `aula_bloqueio`.
   - `delete-aula` → cancela o `aula_bloqueio` (libera as quadras).

D. **Kernel de conflito unificado** passa a mesclar **3 fontes**: `events_scheduled` (só `OUTRO`)
   + `mensalista_plano` + `aula_bloqueio`. `source: 'EVENT' | 'MENSALISTA' | 'AULA'`.

E. **`events_scheduled` reduzido a `OUTRO`.** `create/update` rejeitam `MENSALISTA`,
   `AULA_BEACH_TENIS`, `AULA_VOLEI`. Enum mantido (dados legados + fonte da migração).

F. **Migração** (`AULA_BEACH_TENIS`/`AULA_VOLEI`) → 1 `aula_bloqueio` por registro, **sem
   `aula_id`** (não há vínculo com uma aula real do `beach-center-bff-aulas`); `modalidade`
   derivada do `event_type` (`AULA_BEACH_TENIS → "Beach Tenis"`, `AULA_VOLEI → "Volei"`);
   idempotente via `source_event_id`.

## Regras de negócio conhecidas

1. CRUD de `mensalista` (create/read/update/delete/list) — ADMIN.
2. CRUD de `mensalista_plano` por mensalista (rota aninhada) — ADMIN.
3. `ativo` do mensalista recalculado de forma lazy na listagem, a partir de `vencimento_fatura`
   (mesmo padrão de `list-alunos` da task 002 / `markPastSchedulingsUnavailable`).
4. Criar um plano confirmado:
   - valida que a quadra existe (e deriva `unit`);
   - rejeita se conflitar com aula/evento/outro plano no mesmo dia-da-semana + janela + quadra
     (`EventConflictError` ou equivalente);
   - marca `available = false` nos `scheduling`s afetados (hoje em diante), como
     `applyEventToSchedulings`.
5. Cancelar/deletar um plano → libera os `scheduling`s afetados que não tenham reserva ativa e
   não estejam bloqueados por outro evento/plano (`releaseEventFromSchedulings`).
6. Slots gerados no futuro (via `list-day-schedulings`, que cria `scheduling` sob demanda e
   consulta `eventConflictService`) já nascem `available = false` se caírem num plano confirmado
   — **de graça**, desde que o `EventConflictService` passe a enxergar os planos.
7. `event_type = 'MENSALISTA'` deixa de ser criado por `events_scheduled`.
8. Migração one-shot dos `events_scheduled` `MENSALISTA` existentes → `mensalista_plano`.

## Perguntas em aberto (para `/speckit-plan` ou confirmação do usuário)

1. **Migração — de onde vem o `mensalista` (pessoa)?** Um `events_scheduled` `MENSALISTA` só tem
   `court`, `unit`, `day_of_week`, `start_time`, `end_time`, `status`, `price` — **não tem
   nome/telefone/vencimento**. Opções: (a) criar 1 `mensalista` placeholder por
   `events_scheduled` MENSALISTA (nome tipo "Mensalista importado — <quadra> <dia> <hora>",
   `ativo = true`, `vencimento_fatura` = fim do mês atual); (b) criar 1 `mensalista` genérico
   "Mensalistas legados" e pendurar todos os planos nele; (c) o usuário fornece uma planilha de
   mapeamento. Quantos registros `MENSALISTA` existem em produção?
2. **Nome/estrutura do status do plano.** `status: 'CONFIRMADO' | 'CANCELADO'` (como
   `events_scheduled`) ou um booleano `cancelado`? Precisa de `PENDENTE`/`SUSPENSO`?
3. **`modalidade`** — string livre ou enum fixo (ex.: `"Beach Tenis" | "Volei"`, como em aulas)?
4. **`equipamentos_proprios`** — só informativo, ou entra em alguma regra (preço, disponibilidade
   de equipamento)? Hoje `reserva` tem lógica de equipamento — o mensalista interage com isso?
5. **`vencimento_fatura` / valor da mensalidade.** Igual à task 002: controle 100% manual, **sem
   integração com `pagamentos`** nesta rodada? Existe um campo de valor da mensalidade
   (`price` do `events_scheduled` era opcional)?
6. **`events_scheduled.event_type`** — remover `'MENSALISTA'` do enum agora (com a migração
   esvaziando esses registros) ou manter deprecated até a separação de aula/campeonato terminar?
7. **`list-day-schedulings` / painel admin.** Hoje o `list-day-schedulings` devolve
   `IExceptionConflict[]` (usado pelo painel "Exceções" do `beach-center-app`). Os planos de
   mensalista devem aparecer nesse mesmo retorno (para o admin ver "este slot está bloqueado por
   mensalista X")? Ou o front terá uma tela própria de mensalistas consumindo `/mensalistas`?
   (o front é fora de escopo, mas a **forma da resposta da API** precisa ser decidida.)
8. **Cancelamento de uma ocorrência pontual** do plano (só numa data específica) — dentro do
   escopo desta task ou adiado (como foi para a aula na task 002)?
9. **`dias` — vocabulário e validação.** String livre ou enum `segunda..domingo`? O
   `events_scheduled` usa `day_of_week` (um **único** dia por registro) — o plano tem `dias[]`
   (vários). O `EventSchedulingImpactService` itera por dia; um plano com N dias gera N
   "aplicações". Confirmar que a modelagem é 1 plano com `dias[]` (e não 1 plano por dia).
10. **Autorização.** ADMIN-only em tudo (como `events_scheduled`)? Algum autoatendimento do
    próprio mensalista? (a interface não tem login, então provavelmente ADMIN-only.)
11. **Soft-delete.** `mongoose-delete` como o resto de `agendamentos` (`court`/`unit`), ou o
    `status = CANCELADO` já cobre o "delete" do plano?
12. **`horaInicio`/`horaFim` como `Date`.** O `events_scheduled` guarda `start_time`/`end_time`
    como **string** `"HH:MM"` (recorrência não tem data). O plano recebeu `Date` na interface —
    provavelmente deve virar string `"HH:MM"` no modelo (só o horário importa, é recorrente).
    Confirmar.

## Impacto arquitetural previsto (Arquitetura Hexagonal, Princípio II)

Tudo em `services/beach-center-bff-agendamentos/src/`:

### Novo domínio `mensalista`
- `domain/models/mensalista.model.ts` — `IMensalista`, `ICreateMensalistaData`, `IUpdateMensalistaData`.
- `domain/models/mensalista-plano.model.ts` — `IMensalistaPlano` + data types.
- `domain/ports/input/` — `mensalista.input-port.ts`, `mensalista-plano.input-port.ts`.
- `domain/ports/output/` — `mensalista-persistence.port.ts`, `mensalista-plano-persistence.port.ts`,
  + port de recálculo lazy de `ativo` (`IRecalculateMensalistasStatusPort`),
  + port de contagem/consulta de planos por quadra/unidade para o conflito.
- `domain/usecases/mensalista/{create,read,update,delete,list}/*.usecase.ts`
  (`list` = recálculo lazy de `ativo`).
- `domain/usecases/mensalista-plano/{create,read,update,delete,list,cancel}/*.usecase.ts`
  (`create` = valida quadra, deriva unit, checa conflito, aplica bloqueio;
   `cancel`/`delete` = libera schedulings).
- `infra/schemas/` — `mensalista.schema.ts`, `mensalista-plano.schema.ts` (Mongoose + `mongoose-delete`).
- `infra/adapters/mensalista/**` e `infra/adapters/mensalista_plano/**` — um adapter por verbo/ação
  (create/read/update/delete/list/recalculate-status/count-by-court/find-confirmed-by-court-unit).
- `applications/dto/` — `mensalista.dto.ts`, `mensalista-plano.dto.ts` (yup).
- `applications/controllers/mensalista/**` e `mensalista_plano/**` — thin.
- `applications/routes/` — `mensalista.route.ts` (`/mensalistas`), `mensalista-plano.route.ts`
  (`/mensalistas/:mensalista_id/planos`), montagem em `routes.ts`.

### Alterações em código existente (`agendamentos`)
- `domain/usecases/shared/event-conflict.service.ts` — passa a consultar **também** os
  `mensalista_plano` confirmados (novo port `IFindConfirmedMensalistaPlanosByCourtUnitPort`),
  unindo os conflitos de `events_scheduled` + `mensalista_plano`.
- `domain/usecases/shared/event-scheduling-impact.service.ts` — generalizar para aceitar a
  "fonte" do bloqueio (evento **ou** plano de mensalista) — ou extrair um serviço compartilhado
  `RecurringSlotImpactService` reutilizado pelos dois. O `releaseEventFromSchedulings` já
  re-checa `eventConflictService.hasConflict`, então após ele enxergar os planos, a liberação
  fica correta automaticamente.
- `domain/models/events-scheduled.model.ts` + `infra/schemas/events-scheduled.schema.ts` +
  `applications/dto/events-scheduled.dto.ts` — remover/deprecar `'MENSALISTA'` do enum
  `EventsScheduledType` (conforme pergunta #6).
- `domain/usecases/events_scheduled/create/create-events-scheduled.usecase.ts` — rejeitar
  `event_type = 'MENSALISTA'` (ou remover a opção do DTO).
- `config/container.ts` — wiring dos ~12 novos usecases + novos adapters + injeção do novo port
  no `EventConflictService`.
- **Script de migração** (one-shot) — `src/scripts/migrate-mensalista-events.ts` (ou equivalente
  ao padrão do repo) convertendo `eventos_agendados` `MENSALISTA` → `mensalista` + `mensalista_plano`.

### Perguntas em aberto — parte AULA

13. **`IAula.hora_inicio`/`hora_fim` são `Date`** (task 002). Quem converte para `"HH:MM"` — o
    `beach-center-bff-aulas` antes de enviar, ou `agendamentos` ao receber? (recomendação: o
    cliente envia `"HH:MM"`, `agendamentos` só valida.)
14. **Falha ao criar o `aula_bloqueio`** (agendamentos fora do ar, conflito): a criação da aula
    faz rollback total (deleta o documento da aula) ou fica num estado "aula sem bloqueio" para
    retry? (recomendação: rollback, como o `agendamentos → pagamentos` faz hoje.)
15. **Conflito ao criar aula** — se a quadra já está bloqueada por mensalista/outro evento/outra
    aula, o `POST /aulas` (task 002) passa a retornar 409/400? Hoje só valida o professor.
16. **Aula existente sem bloqueio** — as aulas já cadastradas no `beach-center-bff-aulas` (task
    002) **não** têm `aula_bloqueio` correspondente. Precisa de um passo de sincronização
    (varre as aulas e cria os bloqueios) além da migração dos `events_scheduled`?
17. **`beach-center-bff-aulas` `env.ts`** — adicionar `AGENDAMENTOS_API_URL` e
    `AGENDAMENTOS_INTERNAL_API_KEY` (hoje o serviço não fala com ninguém).
18. **Rota interna** em `agendamentos` — `/aula-bloqueios` (`internalApiKeyMiddleware`), separada
    das rotas ADMIN de `/mensalistas`.
19. **`modalidade` da aula** — o `IAula` da task 002 tem `modalidade: string` (livre). Bate com
    o `aula_bloqueio.modalidade` livre. OK.

## Fora de escopo (a confirmar)

- Integração com `beach-center-bff-pagamentos` para a mensalidade (mesma linha da task 002).
- Separação de **campeonato** de `events_scheduled` (task futura — sobra `OUTRO` + `CAMPEONATO`
  quando existir; hoje campeonato nem está no enum).
- Front-end (`beach-center-app`) — telas de mensalista/aula. A task entrega só a API.
- Cancelamento de ocorrência pontual de um plano/bloqueio (pergunta #8).
- `beach-center-bff-campeonatos` — repo vazio, sem relação com esta task.
