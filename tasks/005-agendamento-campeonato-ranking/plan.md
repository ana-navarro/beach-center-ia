# Plano — 005 Agendamento de Campeonato e Ranking (separação final de `events_scheduled`)

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.
> Todas as perguntas de regra de negócio do `context.md` estão fechadas (3 rodadas de perguntas).

## ⚠️ Revisão arquitetural (2026-09-09, feita durante o `/speckit-implement`, após a Fase 4)

O usuário pediu, no meio da implementação, que `campeonato`/`ranking` seguissem o **mesmo padrão
de integração da aula** (task 004): o MS dono do domínio de negócio empurra o bloqueio de quadra
pra `agendamentos` via rota **interna** (`x-api-key`) — `agendamentos` nunca chama o outro serviço
de volta. Isso **reverte parte do D2/D3/D6 abaixo** e do que tinha sido feito nas Fases 2 e 3:

- **Removido de `agendamentos` inteiramente** (não existe mais nesta base): CRUD de `campeonato`
  (entidade-mãe), CRUD de `ranking` (entidade-mãe), e todo o domínio de `partida`
  (`IDupla`/`ISolo`/`IEquipe`/`IPartida`, `partida-composicao.validator`, `partida.dto`). Esses
  dados (nome/modalidade do campeonato, participantes, chaveamento) passam a ser propriedade do
  **MS de campeonatos** (`services/beach-center-bff-campeonatos`) — que continua **vazio nesta
  task** (construí-lo é fora de escopo aqui, exatamente como já era); a integração é só o contrato
  que `agendamentos` expõe pra ele consumir no futuro.
- **`campeonato_agendamento` virou o equivalente exato de `aula_bloqueio`**: só dados de
  agendamento (`id_campeonato` — referência opaca —, `data`, `hora_inicio`, `hora_fim`, `court`,
  `unit`, `modalidade`, `status`). Sem `partida`.
- **Rota deixou de ser ADMIN/aninhada e virou interna/plana**: `/campeonato-agendamentos`
  (`internalApiKeyMiddleware`, mesmo padrão de `/aula-bloqueios`) — não mais
  `/campeonatos/:id_campeonato/agendamentos` com Firebase/`requireRole('ADMIN')`.
- **`quantidade` substitui `partidas[]`** no `create-bulk`: como não guardamos participantes, o
  contrato agora pede só "quantos slots alocar"; os N registros voltam na ordem determinística do
  `SlotAllocator` e o MS de campeonatos casa por índice com as partidas que ele mesmo controla.
- **`usuario_nome` da troca de dia vem no corpo**, não de `req.databaseUser` — quem chama é o MS de
  campeonatos (rota interna, sem token Firebase), que sabe qual admin da UI dele disparou a ação.
- **O que NÃO mudou**: toda a lógica de agendamento em si — alocação de slots, bloqueio
  assimétrico (D7), fluxo de conflito com `cancelar_conflitos` (D6, revisado no implement pra 1
  usecase só — ver Fase 4), troca de dia com auditoria (D8), delete/delete-bulk (D9), e o kernel
  (`EventConflictService` com a 4ª fonte `CAMPEONATO`) — está tudo igual, só sem o dado de partida.
- Ranking (Fase 5, ainda não implementada) segue o mesmo molde: `ranking_agendamento` também vira
  "só agendamento" (sem partida), rota interna, `id_ranking` opaco. O item D5/D11 (protocolo único
  + comprovante) permanece — isso é sobre o PAGAMENTO da partida de ranking, não sobre quem é dono
  do dado de campeonato/ranking, então não muda com esta revisão.

As seções D2/D3/D6 abaixo (texto original do `/speckit-plan`) ficam como **registro histórico** da
decisão original — não refletem mais o código. O checklist de cada fase tem a nota de revisão nos
itens afetados.

### Atualização 2 (2026-09-09, sessão seguinte) — o MS de campeonatos deixou de ficar vazio

A revisão acima (feita durante a Fase 4/5) dizia que `services/beach-center-bff-campeonatos`
"continua vazio nesta task; construí-lo é fora de escopo". **Isso não é mais verdade** — o usuário
pediu explicitamente pra construir o lado que faltava, e o repo (antes só `package.json`) ganhou:

1. **Bootstrap completo do serviço** (mesma estrutura de `beach-center-bff-aulas`):
   `tsconfig.json`, `eslint.config.mjs`, `jest.config.ts`, `nodemon.json`, `index.js`, `.gitignore`,
   `main.ts`, `config/{env,firebase,container}.ts`, `domain/errors.ts`, camada de `auth`/`user`
   idêntica à de `aulas`.
2. **CRUD de `campeonato`/`ranking`/`partida`** — exatamente o que a Fase 2/3 original tinha
   implementado dentro de `agendamentos` e depois foi revertido de lá; agora vive no lugar certo
   (o MS dono do dado de negócio), com `IPartida`/`IDupla`/`ISolo`/`IEquipe` (D3, C-3-bis) e
   `assertPartidaComposicaoValida` reconstruídos aqui.
3. **A integração que a revisão 1 apenas *previu* o contrato pra, mas nunca implementou do lado
   de quem chama**: `beach-center-bff-campeonatos` agora de fato chama as rotas internas
   (`x-api-key`) de `agendamentos` — `POST /campeonatos/:id/agendar` e `POST /rankings/:id/agendar`
   (empurra bloqueio das partidas pendentes via `create-bulk`, liga cada partida ao `id_agendamento`
   retornado, por índice), liberação automática do bloqueio ao cancelar campeonato/ranking (`delete-
   bulk` sem `ids`) ou uma partida específica (`delete-bulk` com `ids: [id_agendamento]`), e
   `PATCH /partidas/:id/trocar-dia` (repassa pra `trocar-dia` de `campeonato-agendamentos`/
   `ranking-agendamentos`, preservando o fluxo de conflito com decisão do lado do campeonato e a
   auditoria — `usuario_nome` vem de `req.databaseUser.name`, autenticado nesse MS).

Ver **Fase 7** (novo, ao final do checklist) para o detalhamento arquivo-a-arquivo. Isso fecha,
finalmente, o "Serviço(s) alvo" da task: **dois repos afetados**, não um só (tabela abaixo
atualizada).

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Serviço | Situação | Mudança |
|---|---|---|
| `services/beach-center-bff-agendamentos` | produção, hexagonal (tasks 001/004) | 2 domínios de agendamento (`campeonato_agendamento`/`ranking_agendamento` — "só bloqueio de quadra", sem CRUD de entidade-mãe nem partida, ver Revisão arquitetural); kernel de conflito/bloqueio generalizado para **matching por data específica** (além do já existente por dia-da-semana) e para **5 fontes**; `ranking_agendamento` ganha protocolo único + comprovante (Google Drive, autocontido). |
| `services/beach-center-bff-campeonatos` | **vazio → bootstrapado nesta task** (Atualização 2) | Dono de `campeonato`/`ranking`/`partida` (CRUD + composição), e do lado cliente da integração com `agendamentos` (`agendar`, `trocar-dia`, liberação ao cancelar) — ver Fase 7. |

`beach-center-app` fica fora — front-end não entra em nenhuma parte desta task (consistente com
tasks 002/003/004).

### Stack (idêntica ao restante de `agendamentos`)

Node + TypeScript, Express, Mongoose/MongoDB, `yup`, `firebase-admin`, `nanoid` (`customAlphabet`
para protocolo, mesmo padrão de `CreateReserveUsecase`), Jest + ts-jest (`coverageThreshold`
global 80%), ESLint flat config com `no-restricted-imports`.

### Como o kernel funciona hoje (base da task, ponto crítico)

- **`EventConflictService`** (`domain/usecases/shared/event-conflict.service.ts`) já mescla 3
  fontes opcionais (`events_scheduled` `OUTRO`, `mensalista_plano`, `aula_bloqueio` — task 004),
  cada uma um `IEventsScheduled`-shape com `day_of_week` + `source`. O matching é **sempre por
  dia-da-semana recorrente** (`normalizeDayOfWeek(blocker.day_of_week) === dayOfWeek`) — não existe
  hoje um conceito de bloqueio **pontual por data específica**. Esse é o gap que esta task precisa
  fechar (campeonato/ranking são por `data`, não recorrentes).
- **`EventSchedulingImpactService`** (`applyEventToSchedulings`/`releaseEventFromSchedulings`)
  também opera só por `day_of_week`: aplica/libera em **todo `scheduling` futuro** daquele dia da
  semana. Um bloqueio pontual não pode usar isso sem adaptação — bloquearia o dia da semana inteiro
  para sempre, não só aquela data.
- **`SchedulingWindowValidator`** (`domain/usecases/scheduling/shared/scheduling-window.validator.ts`)
  já resolve o horário de funcionamento por unidade (`resolveOperatingHours`: `22:00` para a
  unidade `6a440a931094fad2f585011b`, `23:00` para as demais) — reaproveitado para a regra de
  bloqueio assimétrica do campeonato (decisão 4).
- **Protocolo de reserva** (`CreateReserveUsecase`): `number: data.number || generateNumericId()`
  com `customAlphabet('0123456789', 10)` — mesmo padrão a reaproveitar em `ranking_agendamento`.
- **Comprovante manual** (`beach-center-bff-pagamentos`): `GoogleDriveFileStorageAdapter` (JWT de
  service account, upload multipart na Drive API v3, fallback para base64 em banco se não
  configurado). Decisão do usuário: **não integrar** com `pagamentos` — `agendamentos` ganha uma
  **cópia própria** desse adapter/porta, com suas próprias env vars.
- **Rota pública por protocolo** já existe: `reserva.route.ts` — `GET /reservas/protocol/:number`
  e `PATCH /reservas/protocol/:number/cancel`, **sem** `authMiddleware`. É o padrão exato a
  replicar para o endpoint público de anexar comprovante do ranking (decisão 26).

### Decisões de design desta etapa (fecham os pontos deixados em aberto no `context.md`)

**D1 — Extensão do kernel para bloqueio por data específica (aditiva, sem quebrar o existente).**
- `domain/models/events-scheduled.model.ts`: `IEventsScheduled` ganha `specific_date?: string`
  (`"YYYY-MM-DD"`). Ausente = bloqueio recorrente (comportamento atual, inalterado).
  `RecurringBlockerSource` (mantém o nome — menor diff) ganha `'CAMPEONATO' | 'RANKING'`.
- `EventConflictService`: construtor ganha 2 ports opcionais a mais (5 fontes no total):
  `findConfirmedCampeonatoAgendamentosPort?`, `findConfirmedRankingAgendamentosPort?`. Dentro de
  `getConflicts`, computa `const localDateKey = target.localDateKey ?? (target.date ?
  getLocalDateKey(target.date) : undefined)` e adiciona um filtro **depois** do filtro de
  dia-da-semana existente: `.filter((blocker) => !blocker.specific_date || blocker.specific_date
  === localDateKey)`. Blockers sem `specific_date` (as 3 fontes de hoje) não são afetados.
- `EventSchedulingImpactService`: `IEventSchedulingImpactInput` ganha `specific_date?: string`.
  Em `applyEventToSchedulings`/`releaseEventFromSchedulings`, quando `specific_date` está
  presente, o filtro de schedulings usa `getLocalDateKey(scheduling.date) === event.specific_date`
  **em vez de** `getLocalDayOfWeek(scheduling.date) === normalizeDayOfWeek(event.day_of_week)`.
  `day_of_week` continua obrigatório no input (segue derivado da `data` só para preencher o campo
  sem uso quando `specific_date` está setado — evita 2 shapes de input paralelos).
- `IExceptionConflictEvent.source` (`day.input-port.ts`) ganha `'CAMPEONATO' | 'RANKING'`.
- **DRAFT nunca entra no kernel**: os ports `find-confirmed-*-by-court-unit` de campeonato só
  retornam agendamentos `status='CONFIRMED'` (igual às demais fontes) — `DRAFT` é
  invisível pro `EventConflictService`/scheduling, por definição (decisão 13).

**D2 — Duas entidades-mãe leves, uma delas com 2 modos.**
```ts
// campeonato.model.ts
type CampeonatoStatus = 'ATIVO' | 'CANCELADO';
interface ICampeonato { id: string; nome: string; modalidade: string; status: CampeonatoStatus; }

// ranking.model.ts
type RankingModo = 'PARTIDAS' | 'GERAL';
interface IRanking {
  id: string; nome: string; modalidade: string; modo: RankingModo;
  data_inicio?: string; data_fim?: string;  // obrigatórios só quando modo === 'GERAL'
  status: 'ATIVO' | 'CANCELADO';
}
```
Um único model `IRanking` com `modo` (em vez de duas coleções) — é a mesma entidade "container",
só muda se ela tem `ranking_agendamento`s filhos (`PARTIDAS`) ou nenhum (`GERAL`, cadastro sem
bloqueio nenhum — decisão 7b). CRUD simples, ADMIN, sem lógica de conflito.

**D3 — `IPartida` corrigida (C-3-bis) — mesmo shape para campeonato e ranking.**
```ts
type TipoParticipante = 'DUPLA' | 'SOLO' | 'EQUIPE';
interface IDupla  { tipo: 'DUPLA';  participante_a: string; tel_participante_a: string; participante_b: string; tel_participante_b: string; }
interface ISolo   { tipo: 'SOLO';   participante: string; tel_participante: string; }
interface IEquipe { tipo: 'EQUIPE'; participantes: string[]; tel_capitao: string; }
type IParticipante = IDupla | ISolo | IEquipe;
interface IPartida { lado_a: IParticipante; lado_b: IParticipante; }
```
Regra de composição: `lado_a.tipo === lado_b.tipo`, validada no DTO (yup `.test()`) **e** no
usecase de create/create-bulk (defesa em profundidade, como `mensalista-plano-conflict.error`).

**D4 — `campeonato_agendamento` (a "partida marcada" do campeonato).**
```ts
type CampeonatoAgendamentoStatus = 'DRAFT' | 'CONFIRMED' | 'CANCELLED';
interface ICampeonatoAgendamento {
  id: string; id_campeonato: string; partida: IPartida;
  data: string; hora_inicio: string; hora_fim: string;   // "YYYY-MM-DD" / "HH:MM"
  court: string; unit: string; modalidade: string;
  status: CampeonatoAgendamentoStatus;
}
```
1 registro = 1 quadra = 1 partida = 1 data (decisão 11). `DRAFT` só existe como resultado do fluxo
de conflito (D6) — não é um valor que a API aceita diretamente no create.

**D5 — `ranking_agendamento` (a "partida marcada" do ranking — só existe quando `modo='PARTIDAS'`).**
```ts
// reserva.model.ts (ALTERAR): IReserveStatus ganha 'waiting_approve'
// type IReserveStatus = 'pending' | 'waiting_approve' | 'approved' | 'rejected' | 'cancelled';
type RankingAgendamentoStatus = IReserveStatus;  // reaproveita o enum, não acopla comportamento

interface IComprovanteRanking {
  file_name: string; mime_type: string; storage: 'google_drive' | 'database';
  drive_file_id?: string; data?: string; view_url?: string; preview_url?: string;
}
interface IRankingAgendamento {
  id: string; id_ranking: string; numero_protocolo: string;   // único GLOBAL (decisão 25)
  partida: IPartida;
  data: string; hora_inicio: string; hora_fim: string;
  court: string; unit: string; modalidade: string;
  status: RankingAgendamentoStatus;
  comprovante?: IComprovanteRanking;
}
```
`IReserveStatus` é um tipo **compartilhado** — a mudança é aditiva (`+ 'waiting_approve'`) e
**nenhum usecase de reserva comum muda de comportamento** (ninguém seta esse valor hoje). Só
`ranking_agendamento` usa.

**D6 — Fluxo de conflito de 2 passos, exclusivo do CAMPEONATO (decisão 13).** Substitui
"create"/"create-bulk" tradicionais — todo campeonato_agendamento (unitário ou em massa) nasce
deste fluxo:

1. **`POST /campeonatos/:id_campeonato/agendamentos/verificar`** (dry-run, não persiste nada).
   Body:
   ```ts
   {
     unit: string; quadras: string[]; datas: string[];        // "YYYY-MM-DD"[]
     hora_inicio: string;                                     // "HH:MM", 1º slot do dia
     duracao_partida_minutos: number;                         // ex.: 60
     modalidade: string;
     partidas: { lado_a: IParticipante; lado_b: IParticipante }[];   // 1..N
   }
   ```
   **Algoritmo de alocação (determinístico — mesmo em `verificar` e `confirmar`):** para cada
   `data` (ordem recebida) × `quadra` (ordem recebida), gera slots sequenciais de
   `duracao_partida_minutos` a partir de `hora_inicio`, parando de gerar quando o próximo
   `hora_fim` ultrapassaria `SchedulingWindowValidator.resolveOperatingHours(unit).end_hour`.
   Preenche as `partidas[]` (ordem recebida) nesses slots, um por um. Se sobrarem partidas sem
   slot → `400` (`"Quadras/datas insuficientes para N partidas"`). Cada slot preenchido é um
   **candidato** `{ partida, data, hora_inicio, hora_fim, court, unit, modalidade }`.
   Para cada candidato, roda **dois tipos de checagem** (sem rejeitar — só coletar):
   - **Bloqueadores**: `EventConflictService.getConflicts` (as 5 fontes: `OUTRO`,
     `mensalista_plano`, `aula_bloqueio`, `ranking_agendamento`, outro `campeonato_agendamento`
     `CONFIRMED`) na janela **assimétrica do campeonato** (D1 + regra de horário do campeonato,
     abaixo).
   - **Reservas ativas**: `scheduling`s daquela quadra+unit+data com reserva ativa
     (`hasActiveReservePort`, o mesmo usado por `EventSchedulingImpactService`).
   Retorna `{ candidatos: [...], conflitos: [{ candidato_index, tipo: 'BLOQUEADOR'|'RESERVA',
   source?, id, detalhe }] }` — **nada é persistido**.

2. **`POST /campeonatos/:id_campeonato/agendamentos/confirmar`** — **mesmo body** de `verificar`
   **+ `cancelar_conflitos: boolean`**. Recalcula os mesmos candidatos (determinístico, sem
   estado de servidor entre as 2 chamadas) e:
   - `cancelar_conflitos: true` → para cada conflito encontrado, cancela/libera a fonte
     correspondente (dispatch por `tipo`/`source` — `cancel-mensalista-plano`,
     `cancel-aula-bloqueio`, `update-events-scheduled` → `CANCELLED`, `cancel-ranking-agendamento`,
     ou `updateReserveAndSchedulingStatus`/liberação de scheduling para reserva comum) e persiste
     **todos** os candidatos como `campeonato_agendamento` `CONFIRMED`, aplicando o bloqueio
     (`applyEventToSchedulings` com `specific_date`).
   - `cancelar_conflitos: false` → **nada é cancelado**; persiste todos os candidatos como
     `DRAFT` (sem chamar `applyEventToSchedulings` — não bloqueia nada).
   - Endpoint único cobre criação **unitária** (1 partida) e **em massa** (N) — resolve os
     "endpoints comuns" (create) e "agendamento em massa" pedidos originalmente com o mesmo
     contrato, coerente com a decisão de prioridade máxima do campeonato.

**D7 — Regra de bloqueio assimétrica (decisão 4), aplicada no `applyEventToSchedulings`/conflito:**
- **CAMPEONATO**: janela efetiva = `hora_inicio` até `max(hora_fim, fechamento_da_unidade se
  hora_fim >= fechamento)`. Concretamente: se `hora_fim < resolveOperatingHours(unit).end_hour`,
  usa `hora_fim`; senão usa `end_hour` da unidade.
- **RANKING**: janela efetiva = `hora_inicio`–`hora_fim` (exata, sem extensão).
- Calculado uma vez no usecase (create-bulk/verificar e change-day) antes de checar conflito e
  antes de aplicar bloqueio — o valor efetivo de `hora_fim` é o que vai pro banco (o `hora_fim`
  "de exibição" da partida continua o original informado, se precisar guardar os dois; decisão de
  implementação: **guardar só o efetivo** em `hora_fim`, mais simples, sem campo duplicado).

**D8 — `change-day` (troca de dia com auditoria) — simétrico nos 2 domínios, conflito assimétrico.**
- `PATCH /.../agendamentos/:id/trocar-dia` — body `{ dia_novo: string; motivo: string;
  cancelar_conflitos?: boolean }` (o último só relevante pro campeonato).
- Ambos: valida `dia_novo` (não passado), `motivo` obrigatório (yup `required`), busca o
  agendamento (só `CONFIRMED`, não `DRAFT`/`CANCELLED` — `409` senão).
- **CAMPEONATO**: reaproveita o mesmo mecanismo de D6 para o **novo** slot (mesma quadra, nova
  data) — se houver conflito e `cancelar_conflitos` não vier `true`, retorna os conflitos sem
  mover nada (`200` com a lista, não muda o registro); se `true` (ou não havia conflito), cancela
  os conflitantes, libera o slot antigo (`releaseEventFromSchedulings` com `specific_date` antigo),
  atualiza `data` no registro, aplica bloqueio no novo slot, grava auditoria.
- **RANKING**: checagem simples (409 se o novo slot já tem conflito — sem fluxo de 2 passos);
  libera o slot antigo, atualiza `data`, aplica bloqueio (janela exata), grava auditoria.
- Auditoria: `{ id, agendamento_id, usuario_nome: req.databaseUser.name, motivo, dia_anterior,
  dia_novo, created_at }` — 1 registro por chamada bem-sucedida, 1 coleção por domínio
  (`campeonato_agendamento_auditorias` / `ranking_agendamento_auditorias` — decisão 3, sem tabela
  unificada com `tipo`).

**D9 — Delete / delete-bulk.**
- `PATCH /:id/delete` — `CONFIRMED → CANCELLED` + `releaseEventFromSchedulings`; `DRAFT →
  CANCELLED` sem tocar em scheduling (nunca bloqueou nada).
- `PATCH /delete-bulk` — body `{ ids?: string[] }`; sem `ids`, cancela **todos** do
  `id_campeonato`/`id_ranking` da rota (decisão 10 — os dois jeitos, via presença/ausência de
  `ids`, sem 2 endpoints).

**D10 — Protocolo único globalmente (decisão 25).** Novo port
`domain/ports/output/protocol-uniqueness.port.ts` — `IIsProtocolNumberTakenPort` (consulta
**ambas** as coleções, `reservas` e `ranking_agendamentos`, por `number`/`numero_protocolo`).
`create`/`create-bulk` de `ranking_agendamento` geram com `customAlphabet('0123456789', 10)`
(mesmo padrão de `CreateReserveUsecase`) num loop `while (await isTaken(candidato)) regenerar`
(colisão de 10 dígitos é desprezível, mas o loop cobre o caso).

**D11 — Comprovante do ranking, autocontido (decisões 23/26).**
- `agendamentos` ganha sua própria cópia mínima de `IFileStoragePort`/`GoogleDriveFileStorageAdapter`
  (mesma técnica JWT + Drive API v3 de `beach-center-bff-pagamentos`, **sem** chamá-lo) — novas
  env vars `GOOGLE_DRIVE_CLIENT_EMAIL`/`GOOGLE_DRIVE_PRIVATE_KEY`/`GOOGLE_DRIVE_FOLDER_ID`,
  opcionais (fallback base64-em-banco se ausentes, igual a `pagamentos`).
- **Rota pública** (sem `authMiddleware`), padrão de `reserva.route.ts`:
  `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/comprovante` — body
  `{ file_name, mime_type, base64 }` (PDF ou imagem — yup valida `mime_type` em
  `['application/pdf','image/png','image/jpeg']`). Usecase: busca por `numero_protocolo` →
  `409` se `status !== 'pending'` (mirror do `submit-manual-payment`, que rejeita reenvio) →
  upload → seta `comprovante` + `status: 'waiting_approve'`.
- Endpoint de aprovar/rejeitar **fora de escopo** (decisão 24) — fica parado em
  `waiting_approve` até task futura.

**D12 — Autorização.** Tudo `authMiddleware` + `requireRole('ADMIN')`, **exceto** a rota de
comprovante acima (pública, decisão 19/26).

**D13 — Migração / `events_scheduled`.** Nenhuma migração (C-1 — nunca existiu
`CAMPEONATO`/`RANKING` em `events_scheduled`). `OUTRO` permanece inalterado, único valor restante
do enum genérico (decisões 17/18).

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras** | 2 repos afetados: `beach-center-bff-agendamentos` e `beach-center-bff-campeonatos` (Atualização 2) | **CONFORME.** `agendamentos` é dono exclusivo do bloqueio de quadra; `beach-center-bff-campeonatos` é dono exclusivo do dado de negócio (campeonato/ranking/partida) e fala com `agendamentos` só via rota interna (`x-api-key`) — nunca o contrário, mesmo padrão de `aula`↔`agendamentos` (task 004). Front não é tocado. O comprovante do ranking **duplica** (não importa) a lógica de `beach-center-bff-pagamentos` — decisão deliberada do usuário para não criar acoplamento entre serviços nesta task; reavaliar centralização numa task futura. |
| **II — Hexagonal / Ports** | repo já hexagonal (tasks 001/004) | **CONFORME.** 4 domínios novos seguem a estrutura padrão (models → ports → usecases → schemas → adapters → dto → controllers → routes). O algoritmo de alocação de slots (D6) vive num usecase de domínio (`domain/usecases/campeonato-agendamento/shared/slot-allocator.ts`), puro, sem I/O — só usa os ports injetados para checar conflito/reserva. Nenhum adapter contém regra de negócio. |
| **III — Test-First / Qualidade** | `coverageThreshold` global 80% | **CONFORME** (cobrado no `/speckit-unit-tests`). Superfície nova grande: kernel (`EventConflictService`/`EventSchedulingImpactService`) precisa de specs que cubram **as 5 fontes** e o novo filtro `specific_date`, sem regressão nas 3 fontes existentes (rede de segurança = specs atuais). Algoritmo de alocação de slots merece specs dedicados (casos: exata, sobra quadra, falta quadra, cruza o fechamento da unidade). |
| **IV — Infra hot-reload** | N/A | Sem mudança de infra além de 3 env vars novas opcionais (`GOOGLE_DRIVE_*`) — refletir em `docker-compose.dev.yml`/`.env.dev.example` (task 003) se o comprovante for testado localmente; opcional pro fallback base64 funcionar sem elas. |

**Resultado: sem violação. Nenhum ERRO de bloqueio.**

## Mapa Arquitetural (Hexagonal)

### `services/beach-center-bff-agendamentos/src/`

```
domain/models/
  campeonato.model.ts                         # ICampeonato + data types
  ranking.model.ts                            # IRanking (modo PARTIDAS|GERAL) + data types
  partida.model.ts                            # IDupla/ISolo/IEquipe/IParticipante/IPartida (D3) — compartilhado pelos 2 domínios de agendamento
  campeonato-agendamento.model.ts             # ICampeonatoAgendamento (D4)
  ranking-agendamento.model.ts                # IRankingAgendamento + IComprovanteRanking (D5)
  campeonato-agendamento-auditoria.model.ts   # troca de dia
  ranking-agendamento-auditoria.model.ts      # troca de dia
  reserva.model.ts                 (ALTERAR)  # IReserveStatus + 'waiting_approve'
  events-scheduled.model.ts        (ALTERAR)  # IEventsScheduled + specific_date?; RecurringBlockerSource + CAMPEONATO/RANKING

domain/ports/input/
  campeonato.input-port.ts                    # CRUD
  ranking.input-port.ts                       # CRUD
  campeonato-agendamento.input-port.ts        # verificar/confirmar, read, update, delete, delete-bulk, list, change-day, list-auditoria
  ranking-agendamento.input-port.ts           # create, create-bulk, read, update, delete, delete-bulk, list, change-day, list-auditoria, attach-proof

domain/ports/output/
  campeonato-persistence.port.ts
  ranking-persistence.port.ts
  campeonato-agendamento-persistence.port.ts       # ICreate(status)/IRead/IUpdate/IList/ISetStatus/IDeleteBulk
  campeonato-agendamento-shared.port.ts            # IFindConfirmedCampeonatoAgendamentosByCourtUnitPort (kernel)
  campeonato-agendamento-auditoria-persistence.port.ts   # ICreate/IList
  ranking-agendamento-persistence.port.ts          # idem + IFindByProtocolPort
  ranking-agendamento-shared.port.ts               # IFindConfirmedRankingAgendamentosByCourtUnitPort (kernel)
  ranking-agendamento-auditoria-persistence.port.ts
  protocol-uniqueness.port.ts                      # IIsProtocolNumberTakenPort (D10 — consulta reservas + ranking_agendamentos)
  file-storage.port.ts                             # IUploadProofPort (cópia local, D11)

domain/usecases/shared/                        (ALTERAR)
  event-conflict.service.ts                    # + 2 ports opcionais (5 fontes); filtro specific_date (D1)
  event-scheduling-impact.service.ts           # + specific_date opcional no input (D1)
  conflict-resolution.service.ts               # NOVO — dispatch de cancelamento por `source` (D6.confirmar)

domain/usecases/campeonato/{create,read,update,delete,list}/*.usecase.ts
domain/usecases/ranking/{create,read,update,delete,list}/*.usecase.ts       # valida data_inicio/data_fim quando modo=GERAL

domain/usecases/campeonato-agendamento/shared/slot-allocator.ts             # D6 — puro, sem I/O
domain/usecases/campeonato-agendamento/shared/partida-composicao.validator.ts  # lado_a.tipo === lado_b.tipo
domain/usecases/campeonato-agendamento/verificar/verificar-campeonato-agendamentos.usecase.ts
domain/usecases/campeonato-agendamento/confirmar/confirmar-campeonato-agendamentos.usecase.ts
domain/usecases/campeonato-agendamento/{read,list,update}/*.usecase.ts
domain/usecases/campeonato-agendamento/delete/{delete,delete-bulk}.usecase.ts
domain/usecases/campeonato-agendamento/change-day/change-day-campeonato-agendamento.usecase.ts
domain/usecases/campeonato-agendamento/list-auditoria/list-campeonato-agendamento-auditoria.usecase.ts

domain/usecases/ranking-agendamento/shared/partida-composicao.validator.ts   # reuso do de campeonato (mover pra domain/usecases/shared/)
domain/usecases/ranking-agendamento/{create,create-bulk,read,list,update}/*.usecase.ts
domain/usecases/ranking-agendamento/delete/{delete,delete-bulk}.usecase.ts
domain/usecases/ranking-agendamento/change-day/change-day-ranking-agendamento.usecase.ts
domain/usecases/ranking-agendamento/list-auditoria/list-ranking-agendamento-auditoria.usecase.ts
domain/usecases/ranking-agendamento/attach-proof/attach-proof-ranking-agendamento.usecase.ts   # D11

domain/ports/input/day.input-port.ts           (ALTERAR)   # IExceptionConflictEvent.source + CAMPEONATO/RANKING
domain/usecases/day/list-day-schedulings/list-day-schedulings.usecase.ts  (ALTERAR)  # propaga source

infra/schemas/
  campeonato.schema.ts
  ranking.schema.ts
  campeonato-agendamento.schema.ts             # subdocumento `partida`
  ranking-agendamento.schema.ts                # subdocumento `partida` + `comprovante`; índice único `numero_protocolo`
  campeonato-agendamento-auditoria.schema.ts
  ranking-agendamento-auditoria.schema.ts

infra/adapters/campeonato/{create,read,update,delete,list}/*.adapter.ts
infra/adapters/ranking/{create,read,update,delete,list}/*.adapter.ts
infra/adapters/campeonato_agendamento/{create,read,update,list,set-status,delete-bulk,find-confirmed-by-court-unit}/*.adapter.ts
infra/adapters/campeonato_agendamento_auditoria/{create,list}/*.adapter.ts
infra/adapters/ranking_agendamento/{create,read,update,list,set-status,delete-bulk,find-confirmed-by-court-unit,find-by-protocol}/*.adapter.ts
infra/adapters/ranking_agendamento_auditoria/{create,list}/*.adapter.ts
infra/adapters/protocol/is-protocol-number-taken/is-protocol-number-taken.adapter.ts   # consulta 2 coleções
infra/adapters/file-storage/google-drive-file-storage.adapter.ts    # cópia local (D11)
infra/adapters/reserva/**                       (sem novo adapter — só o type IReserveStatus muda)

applications/dto/
  campeonato.dto.ts
  ranking.dto.ts                                # valida data_inicio/data_fim quando modo=GERAL
  partida.dto.ts                                # yup union discriminada por `tipo`; .test() lado_a.tipo===lado_b.tipo
  campeonato-agendamento-bulk.dto.ts             # verificar/confirmar (D6)
  campeonato-agendamento.dto.ts                  # update (modalidade/partida), change-day (dia_novo/motivo/cancelar_conflitos), delete-bulk
  ranking-agendamento.dto.ts                     # create/create-bulk/update/change-day/delete-bulk
  ranking-agendamento-comprovante.dto.ts         # file_name/mime_type/base64 (D11)

applications/controllers/campeonato/**, ranking/**, campeonato_agendamento/**, ranking_agendamento/**   # thin
applications/controllers/ranking_agendamento/attach-proof/attach-proof.controller.ts   # extrai :numero_protocolo, sem req.databaseUser

applications/routes/
  campeonato.route.ts                           # /campeonatos          (ADMIN)
  ranking.route.ts                               # /rankings             (ADMIN)
  campeonato-agendamento.route.ts                # /campeonatos/:id_campeonato/agendamentos  (ADMIN) — verificar/confirmar/read/list/update/delete/delete-bulk/trocar-dia/auditoria
  ranking-agendamento.route.ts                   # /rankings/:id_ranking/agendamentos  (ADMIN) — create/create-bulk/read/list/update/delete/delete-bulk/trocar-dia/auditoria
  ranking-agendamento-protocol.route.ts          # /ranking-agendamentos/protocolo/:numero_protocolo/comprovante  (PÚBLICA — D11/D12)
  routes.ts                          (ALTERAR)   # monta as 5 rotas novas

config/
  env.ts                             (ALTERAR)   # + GOOGLE_DRIVE_CLIENT_EMAIL/PRIVATE_KEY/FOLDER_ID (opcionais)
  container.ts                       (ALTERAR)   # ~35 usecases + adapters novos; EventConflictService recebe os 2 ports novos; EventSchedulingImpactService sem mudança de assinatura de wiring
```

## Checklist de Implementação

> Ordem: kernel (D1) → entidades-mãe → partida (D3, compartilhada) → campeonato_agendamento
> (D4/D6/D7/D8/D9) → ranking_agendamento (D5/D7/D8/D9/D10/D11) → aplicação/rotas → container.

### Fase 1 — Kernel: suporte a bloqueio por data específica (D1) — ✅ IMPLEMENTADO

- [x] `domain/models/events-scheduled.model.ts` — `IEventsScheduled.specific_date?: string`;
      `RecurringBlockerSource` `+ 'CAMPEONATO' | 'RANKING'`
- [x] `domain/usecases/shared/event-conflict.service.ts` — `localDateKey` derivado de `date`
      quando ausente; filtro `specific_date` pós dia-da-semana. **Desvio do plano:** os 2 ports
      opcionais (`findConfirmedCampeonatoAgendamentosPort`/`findConfirmedRankingAgendamentosPort`)
      **não** foram adicionados ao construtor nesta fase — as interfaces
      `campeonato-agendamento-shared.port.ts`/`ranking-agendamento-shared.port.ts` só nascem nas
      Fases 4/5; adicionar parâmetros tipados para ports inexistentes não compila. O mecanismo de
      filtro por `specific_date` já está pronto e testado (backward-compatible, 0 fontes hoje
      setam `specific_date`); a wiring dos 2 ports novos entra junto com os adapters reais na
      Fase 4/5, sem precisar tocar aqui de novo.
- [x] `domain/usecases/shared/event-scheduling-impact.service.ts` —
      `IEventSchedulingImpactInput.specific_date?: string`; `matchesEventDay` (helper privado)
      centraliza o branch (pontual por `getLocalDateKey` vs. recorrente por `day_of_week`) usado
      em `applyEventToSchedulings` e `releaseEventFromSchedulings`
- [x] `domain/ports/input/day.input-port.ts` — `IExceptionConflictEvent.source` `+
      'CAMPEONATO' | 'RANKING'`
- [x] `domain/usecases/day/list-day-schedulings/list-day-schedulings.usecase.ts` — **nenhuma
      alteração necessária**: `source: event.source ?? "EVENT"` já é genérico e herda o tipo mais
      amplo transitivamente (confirmado via `tsc --noEmit`)
- [x] `tsc --noEmit` limpo; `eslint` limpo nos 5 arquivos; suíte completa **1001/1001** verde
      (sem nenhum spec novo ou alterado — mudança 100% aditiva/retrocompatível, confirmado pelas
      specs existentes de `event-conflict.service`, `event-scheduling-impact.service` e
      `list-day-schedulings` passando sem modificação)

### Fase 2 — Entidades-mãe `campeonato` / `ranking` (D2) — ❌ REVERTIDO (ver Revisão arquitetural)

> Tudo o que estava marcado `[x]` aqui foi **implementado e depois removido** de `agendamentos` —
> `campeonato`/`ranking` (entidade-mãe) passaram a ser propriedade do MS de campeonatos. Nenhum
> arquivo desta fase existe mais nesta base. Mantido abaixo só como registro histórico.

- [x] `domain/models/campeonato.model.ts`, `domain/models/ranking.model.ts` — `status:
      'ATIVO'|'CANCELADO'` (não `deleted` boolean — consistente com o padrão de
      `aula_bloqueio`/`mensalista_plano`, não o de `mensalista`/`court`/`unit`)
- [x] `domain/ports/input/{campeonato,ranking}.input-port.ts`
- [x] `domain/ports/output/{campeonato,ranking}-persistence.port.ts` — inclui
      `ISetCampeonatoStatusPort`/`ISetRankingStatusPort` (delete = soft-cancel via status)
- [x] `domain/usecases/campeonato/{create,read,update,delete,list}/*.usecase.ts` — CRUD simples
- [x] `domain/usecases/ranking/{create,read,update,delete,list}/*.usecase.ts` +
      `domain/usecases/ranking/shared/ranking-modo.validator.ts`
      (`assertRankingModoConsistency`) — valida `data_inicio < data_fim` quando `modo==='GERAL'`;
      rejeita `data_inicio`/`data_fim` quando `modo==='PARTIDAS'`; `update` relê o `modo`
      existente (imutável) antes de revalidar
- [x] `infra/schemas/{campeonato,ranking}.schema.ts`
- [x] `infra/adapters/campeonato/**`, `infra/adapters/ranking/**` (5 cada: create/read/update/
      set-status/list)
- [x] `applications/dto/{campeonato,ranking}.dto.ts` — DTO só valida formato (regex `YYYY-MM-DD`,
      `oneOf` do `modo`); a regra de consistência `modo`×datas fica só no usecase (evita 2 fontes
      de verdade divergentes)
- [x] `applications/controllers/{campeonato,ranking}/**` — thin
- [x] `applications/routes/{campeonato,ranking}.route.ts` — ADMIN (`authMiddleware` +
      `requireRole('ADMIN')`, idêntico a `/mensalistas`)
- [x] `applications/routes/routes.ts` — monta `/campeonatos`, `/rankings`
- [x] `config/container.ts` — wiring completo (10 adapters + 10 usecases)
- [x] `container.spec.ts` (+10 chaves) + `routes.spec.ts` (`KNOWN_PREFIXES` + 2 `it()` novos +
      contagem 53→63) atualizados
- [x] `tsc --noEmit` limpo; `eslint` limpo; suíte completa **1003/1003** verde (sem specs novos
      além da atualização dos 2 specs de wiring acima — cobertura dedicada dos usecases/adapters
      novos fica para `/speckit-unit-tests`)

### Fase 3 — Partida compartilhada (D3) — ❌ REVERTIDO (ver Revisão arquitetural)

> Idem à Fase 2: `partida.model.ts`, `partida-composicao.validator.ts` e `partida.dto.ts` foram
> **removidos** de `agendamentos`. Participantes/chaveamento são dado do MS de campeonatos —
> validar a composição (dupla×dupla etc.) é responsabilidade dele agora. Mantido abaixo como
> registro histórico.

- [x] `domain/models/partida.model.ts` — `IDupla`/`ISolo`/`IEquipe`/`IParticipante`/`IPartida`
- [x] `domain/usecases/shared/partida-composicao.validator.ts` — `assertPartidaComposicaoValida`
      (nome final; `assertMesmoTipo` do plano virou este nome, mais descritivo) → `InvalidInputError`
      se `lado_a.tipo !== lado_b.tipo`
- [x] `applications/dto/partida.dto.ts` — `yup.lazy` discriminado por `tipo` (`DUPLA`/`SOLO`/
      `EQUIPE`, cada um com seu shape) + `.test()` cruzado de composição no schema `partidaDTO`.
      **Ajuste de robustez**: o teste de composição só compara `tipo`×`tipo` quando **ambos**
      estão presentes — senão a falta de `tipo` ficava mascarada pela mensagem genérica de
      "composição inválida" em vez de "tipo é obrigatório" (verificado manualmente com um script
      descartável rodando os 3 tipos × mismatch × tipo ausente × equipe com 1 membro só — todos os
      casos passaram após o ajuste; script removido, não faz parte do repo — cobertura formal via
      Jest fica para `/speckit-unit-tests`, conforme o fluxo).

### Fase 4 — `campeonato_agendamento` (D4, D6, D7, D8, D9) — ✅ IMPLEMENTADO (revisado pós-Fase 4)

> **Revisão arquitetural aplicada aqui** (ver seção no topo do arquivo): removido o campo
> `partida` do modelo/DTOs/usecases; `create-bulk` passou a receber `quantidade: number` em vez de
> `partidas: [...]`; rota saiu de ADMIN/aninhada (`/campeonatos/:id_campeonato/agendamentos`) para
> **interna/plana** (`/campeonato-agendamentos`, `internalApiKeyMiddleware`); `change-day` passou a
> receber `usuario_nome` no corpo em vez de extrair de `req.databaseUser`. Todo o resto (algoritmo
> de alocação, bloqueio assimétrico, fluxo de conflito, auditoria) descrito abaixo continua válido
> — só o transporte/autenticação e a ausência de dado de partida mudaram. Refeito e reverificado
> (tsc/eslint/jest limpos + smoke test manual) depois da revisão.

> **Desvio de design importante (D6 revisado durante o implement):** em vez de 2 usecases/rotas
> separadas (`verificar` dry-run + `confirmar`), implementei **1 usecase único** (`create-bulk`)
> que reaproveita um padrão **já existente e testado** no próprio repo:
> `CloseDateUsecase`/`CloseDateReserveConflictError` (`day/close-date`). Mecânica: a MESMA chamada,
> sem `cancelar_conflitos`, detecta conflito → lança `CampeonatoAgendamentoConflictError` (409,
> carrega a lista) **sem persistir nada**; o admin resubmete a **mesma** chamada com
> `cancelar_conflitos: true` (cancela e confirma) ou `false` (cria como `DRAFT`). Elimina a
> necessidade de determinismo entre 2 chamadas separadas (é a mesma computação, na mesma
> chamada) e segue exatamente a convenção já usada no serviço para esse exato problema (dry-run
> "embutido" como erro 409 + resubmissão) — por isso **não existem** os arquivos
> `verificar-campeonato-agendamentos.usecase.ts`/`confirmar-campeonato-agendamentos.usecase.ts`
> do plano original; existe só `create-bulk/create-bulk-campeonato-agendamentos.usecase.ts`.
> Mesmo padrão aplicado ao `change-day` (D8).

- [x] `domain/models/campeonato-agendamento.model.ts` + `campeonato-agendamento-auditoria.model.ts`
- [x] `domain/ports/input/campeonato-agendamento.input-port.ts` — `ICreateBulkCampeonatoAgendamentosUseCase`
      (cobre unitário e em massa), read/list/update/delete/delete-bulk/change-day/list-auditoria
- [x] `domain/ports/output/campeonato-agendamento-persistence.port.ts` — create-many (status
      DRAFT/CONFIRMED), read, update, list, set-status, **change-data** (só `data`, p/ troca de
      dia), delete-bulk (por `id_campeonato` e/ou lista de ids, devolve o status ORIGINAL
      pré-cancelamento p/ o usecase saber o que precisa liberar scheduling)
- [x] `domain/ports/output/campeonato-agendamento-shared.port.ts` —
      `IFindConfirmedCampeonatoAgendamentosByCourtUnitPort` (só `CONFIRMED`, `specific_date` =
      `data`, `source: 'CAMPEONATO'`, `event_type: 'OUTRO'` placeholder — mesma técnica de
      `aula_bloqueio` reaproveitando um valor existente do enum em vez de o alargar)
- [x] `domain/ports/output/campeonato-agendamento-auditoria-persistence.port.ts`
- [x] `domain/usecases/campeonato-agendamento/shared/slot-allocator.ts` — algoritmo de D6 (puro,
      recebe `SchedulingWindowValidator` injetado). **Refinamento de D7 aplicado aqui**: a
      **última** partida de cada quadra/dia tem `hora_fim` estendido até o fechamento da unidade
      (jogo tardio "estoura" o previsto) — as demais mantêm o fim exato da duração informada;
      verificado manualmente (script descartável, removido): fechamento 22h, 2 partidas de 60min
      a partir de 20h → 20:00–21:00 e 21:00–22:00 (a 2ª, última, já bate com o fechamento);
      alocação insuficiente rejeita com `InvalidInputError` claro.
- [x] `domain/usecases/campeonato-agendamento/shared/campeonato-agendamento-conflict.error.ts` —
      `CampeonatoAgendamentoConflictError extends ConflictError` (409), espelha
      `CloseDateReserveConflictError`
- [x] `domain/usecases/shared/conflict-resolution.service.ts` — dispatch de cancelamento
      **reaproveitando os usecases existentes** (`IDeleteEventsScheduledUseCase`,
      `ICancelMensalistaPlanoUseCase`, `ICancelAulaBloqueioUseCase`, `IDeleteReserveUseCase` — este
      último com `skipCancellationTimeValidation: true`, mesmo padrão de `CloseDateUsecase`) em vez
      de reimplementar cancelamento/estorno; **`RANKING` como port opcional** (interface local
      `ICancelRankingAgendamentoUseCase`, ainda não injetado — Fase 5 substitui pela real e passa a
      injetar); **`CAMPEONATO`** (conflito com outro campeonato_agendamento já confirmado) resolvido
      reaproveitando o próprio `DeleteCampeonatoAgendamentoUsecase` deste domínio
- [x] `domain/usecases/campeonato-agendamento/create-bulk/create-bulk-campeonato-agendamentos.usecase.ts`
      — substitui verificar+confirmar (ver desvio acima); deriva `unit` de `quadras[0]` (valida que
      todas pertencem à mesma unidade — não é mais um campo de entrada); valida composição de
      partida; aloca; detecta conflitos (bloqueadores via `EventConflictService` + reservas ativas
      via `findSchedulingsByCourtUnitFromDatePort`+`findActiveReservesForSchedulingsPort`); decide
      lançar erro / `CONFIRMED`+resolve / `DRAFT`
- [x] `domain/usecases/campeonato-agendamento/{read,list,update}/*.usecase.ts` — `update` só
      `modalidade`/`partida` (revalida composição se `partida` vier)
- [x] `domain/usecases/campeonato-agendamento/delete/delete-campeonato-agendamento.usecase.ts` —
      `CONFIRMED` → `CANCELLED` + release; `DRAFT` → `CANCELLED` direto
- [x] `domain/usecases/campeonato-agendamento/delete/delete-bulk-campeonato-agendamentos.usecase.ts`
      — por `id_campeonato` (todos) e/ou lista de `ids`; só libera scheduling dos que eram
      `CONFIRMED`
- [x] `domain/usecases/campeonato-agendamento/change-day/change-day-campeonato-agendamento.usecase.ts`
      — D8: só `CONFIRMED`; **não** re-executa o slot-allocator (troca só `data`; quadra/horário
      ficam intactos — fiel a "atualizar somente o dia do campeonato"); mesmo padrão de conflito
      409 do create-bulk (aqui sem estado `DRAFT` — só `cancelar_conflitos: true` resolve e move,
      qualquer outro valor lança o conflito); motivo obrigatório; auditoria gravada só em caso de
      sucesso
- [x] `domain/usecases/campeonato-agendamento/list-auditoria/*.usecase.ts` — filtra por
      `agendamento_id` e/ou `id_campeonato` (denormalizado na auditoria, evita join)
- [x] `infra/schemas/campeonato-agendamento.schema.ts` — `partida` como `Mixed` (mesma técnica de
      `equipment` em `reserva.schema.ts` — união discriminada validada no DTO/domain, não no
      schema); índices `{court,unit,data,status}`, `id_campeonato`
- [x] `infra/schemas/campeonato-agendamento-auditoria.schema.ts`
- [x] `infra/adapters/campeonato_agendamento/**` (8: create-many, read, list, update, set-status,
      change-data, delete-bulk, find-confirmed-by-court-unit — 1 a mais que a estimativa do plano,
      por causa do `change-data`/`delete-bulk` dedicados) + `infra/adapters/campeonato_agendamento_auditoria/**` (2)
- [x] `applications/dto/campeonato-agendamento-bulk.dto.ts` (create-bulk, reaproveita `partidaDTO`)
      + `applications/dto/campeonato-agendamento.dto.ts` (update/change-day/delete-bulk)
- [x] `applications/controllers/campeonato_agendamento/**` — thin; `create-bulk`/`change-day`
      capturam `CampeonatoAgendamentoConflictError` explicitamente (mesmo padrão de
      `close-date.controller.ts`) antes de cair no `handleHttpError` genérico (que só serializa
      `{message}`, não os campos extras); `change-day` extrai `usuario_nome` de
      `req.databaseUser?.name`
- [x] `applications/routes/campeonato-agendamento.route.ts` — `Router({mergeParams:true})`, ADMIN;
      montada em `routes.ts` como `/campeonatos/:id_campeonato/agendamentos`, **antes** de
      `/campeonatos` (mesmo motivo de `/mensalistas/:mensalista_id/planos` vir antes de
      `/mensalistas`); `/auditoria` e `/delete-bulk` (segmentos literais) registrados antes de
      `/:id` pra não colidir
- [x] `config/container.ts` — wiring completo (8 adapters + 2 adapters de auditoria + 8 usecases +
      `SlotAllocator` + `ConflictResolutionService`); `EventConflictService` recebe o 4º port
      (`findConfirmedCampeonatoAgendamentosByCourtUnitAdapter`) — fecha o desvio documentado na
      Fase 1
- [x] `container.spec.ts` (+8 chaves) + `routes.spec.ts` (`KNOWN_PREFIXES` + novo `it()` do
      sub-roteador aninhado + contagem 63→71) atualizados
- [x] `tsc --noEmit` limpo; `eslint` limpo; suíte completa **1004/1004** verde (sem specs novos
      além da atualização dos 2 specs de wiring — cobertura dedicada do `slot-allocator`/
      `create-bulk`/`change-day` fica para `/speckit-unit-tests`). Lógica central (alocação de
      slots incl. extensão até o fechamento, e as 4 combinações de conflito × decisão do
      create-bulk) verificada manualmente com um script descartável (removido, não ficou no repo).

### Fase 5 — `ranking_agendamento` (D5, D7, D8, D9, D10, D11) — ✅ IMPLEMENTADO (revisado)

> **Segue a mesma revisão arquitetural da Fase 4** (ver seção no topo): sem `partida`; rota
> **interna/plana** (`/ranking-agendamentos`, `internalApiKeyMiddleware`) em vez de
> `/rankings/:id_ranking/agendamentos` ADMIN; `create`/`create-bulk` consolidados num usecase só
> (`quantidade`, mesma lógica de `campeonato_agendamento`); `usuario_nome` da troca de dia vem no
> corpo. A rota pública de comprovante (D11/D26) continua igual ao planejado — isso não muda com a
> revisão (é inerentemente user-facing, independente de quem é dono do campeonato/ranking).
>
> **Conflito é 409, não 400** (ajuste em relação ao texto original abaixo): mesma razão de
> `AulaBloqueioConflictError` — rota interna cross-service, o client (MS de campeonatos) distingue
> "conflito" de "payload inválido" pelo status HTTP.
>
> **`IReserveStatus` NÃO foi alargado** (ao contrário do item abaixo) — widening quebrava o schema
> Mongoose de `reserva` (mais estrito, `exactOptionalPropertyTypes`). Criei
> `RankingAgendamentoStatus` como tipo **independente** em `ranking-agendamento.model.ts` (mesmos
> valores + `'waiting_approve'`), sem acoplar aos usecases de reserva comum — solução melhor que a
> alternativa cogitada no plano original, sem nenhum efeito colateral no domínio de reserva.
>
> **`SlotAllocator` movido para `domain/usecases/shared/`** (era só de campeonato) — reaproveitado
> por ranking com um novo parâmetro `extendLastSlotToClosing: boolean` (`true` p/ campeonato,
> `false` p/ ranking — implementa a assimetria de bloqueio, D7, no mesmo algoritmo em vez de
> duplicá-lo).

- [x] `domain/models/ranking-agendamento.model.ts` — `IRankingAgendamento`, `IComprovanteRanking`,
      `RankingAgendamentoStatus` (tipo próprio, não `IReserveStatus` alargado — ver nota acima)
- [x] `domain/models/ranking-agendamento-auditoria.model.ts`
- [x] `domain/ports/input/ranking-agendamento.input-port.ts` — `ICreateBulkRankingAgendamentosUseCase`
      (unitário via `quantidade:1` + em massa, mesmo contrato do campeonato) +
      read/list/update/delete/delete-bulk/change-day/list-auditoria/attach-proof
- [x] `domain/ports/output/ranking-agendamento-persistence.port.ts` — create-many, read,
      **find-by-protocol**, list, update, set-status, change-data, **set-comprovante**, delete-bulk
- [x] `domain/ports/output/ranking-agendamento-shared.port.ts` —
      `IFindConfirmedRankingAgendamentosByCourtUnitPort` (`source: 'RANKING'`, só `status !==
      'cancelled'` — não há um "CONFIRMED" separado aqui, o bloqueio vale desde a criação)
- [x] `domain/ports/output/ranking-agendamento-auditoria-persistence.port.ts`
- [x] `domain/ports/output/protocol-uniqueness.port.ts` — `IIsProtocolNumberTakenPort` (D10)
- [x] `domain/ports/output/file-storage.port.ts` — `IFileStoragePort`/`IUploadProofInput` (D11)
- [x] `domain/usecases/shared/slot-allocator.ts` — **movido** de `campeonato-agendamento/shared/`
      (era `IAllocatedCampeonatoSlot`, agora `IAllocatedSlot`); `create-bulk` do campeonato
      atualizado pra importar do novo local e passar `extendLastSlotToClosing: true`
- [x] `domain/usecases/ranking-agendamento/shared/ranking-agendamento-conflict.error.ts` — 409
      (não 400 — ver nota acima)
- [x] `domain/usecases/ranking-agendamento/create-bulk/create-bulk-ranking-agendamentos.usecase.ts`
      — deriva `unit` de `quadras[0]`; aloca com `extendLastSlotToClosing: false`; checa
      conflito **atômico** (bloqueadores + reservas ativas — se qualquer slot conflitar, rejeita o
      lote inteiro, `RankingAgendamentoConflictError`, nada persistido); gera `numero_protocolo`
      único por registro (loop com `IIsProtocolNumberTakenPort`, mesmo padrão de
      `CreateReserveUsecase`); `status: 'pending'`; aplica bloqueio (janela exata) em todos
- [x] `domain/usecases/ranking-agendamento/{read,list,update}/*.usecase.ts` — `update` só
      `modalidade`
- [x] `domain/usecases/ranking-agendamento/delete/{delete,delete-bulk}-ranking-agendamentos.usecase.ts`
      — `status='cancelled'`; libera scheduling só se não estava já `cancelled`
- [x] `domain/usecases/ranking-agendamento/change-day/change-day-ranking-agendamento.usecase.ts`
      — D8: conflito simples 409 no novo slot (bloqueadores + reservas ativas, mesma checagem do
      create-bulk); libera antigo; bloqueia novo (janela exata); auditoria só em caso de sucesso
- [x] `domain/usecases/ranking-agendamento/list-auditoria/*.usecase.ts`
- [x] `domain/usecases/ranking-agendamento/attach-proof/attach-proof-ranking-agendamento.usecase.ts`
      — busca por `numero_protocolo`; `409` (`ConflictError`) se `status !== 'pending'`; upload;
      seta `comprovante` + `status: 'waiting_approve'` atomicamente (1 update)
- [x] `infra/schemas/ranking-agendamento.schema.ts` — **sem** subdoc `partida` (removido da
      revisão); `comprovante` como subschema tipado; índice **único** `numero_protocolo`
- [x] `infra/schemas/ranking-agendamento-auditoria.schema.ts`
- [x] `infra/adapters/ranking_agendamento/**` (9: create-many, read, find-by-protocol, list,
      update, set-status, change-data, set-comprovante, delete-bulk) +
      `infra/adapters/ranking_agendamento/find-confirmed-by-court-unit/**` (kernel) +
      `infra/adapters/ranking_agendamento_auditoria/**` (2)
- [x] `infra/adapters/protocol/is-protocol-number-taken/is-protocol-number-taken.adapter.ts` —
      consulta `reservas.number` **e** `ranking_agendamentos.numero_protocolo` em paralelo
- [x] `infra/adapters/file-storage/google-drive-file-storage.adapter.ts` — cópia local (D11),
      técnica idêntica à de `beach-center-bff-pagamentos` (JWT service-account + Drive API v3),
      sem importar de lá; fallback base64-em-banco se as env vars não estiverem configuradas
- [x] `applications/dto/ranking-agendamento-bulk.dto.ts` (create-bulk) +
      `applications/dto/ranking-agendamento.dto.ts` (update/change-day/delete-bulk) +
      `applications/dto/ranking-agendamento-comprovante.dto.ts` (mime_type `oneOf`
      `['application/pdf','image/png','image/jpeg']`)
- [x] `applications/controllers/ranking_agendamento/**` — thin; **internas** (não ADMIN — ver
      revisão arquitetural)
- [x] `applications/controllers/ranking_agendamento/attach-proof/attach-proof.controller.ts` —
      rota pública, sem `req.databaseUser` nem `internalApiKeyMiddleware`
- [x] `applications/routes/ranking-agendamento.route.ts` — **1 router só, flat**
      (`/ranking-agendamentos`), misturando `internalApiKeyMiddleware` por rota (CRUD) com a rota
      pública do comprovante — mesmo padrão de `reserva.route.ts` (middleware por rota, não
      `.use()` geral). **Não existe** `ranking-agendamento-protocol.route.ts` separado (consolidado
      num arquivo só, já que a rota interna também virou flat)
- [x] `config/env.ts` — `+ GOOGLE_DRIVE_CLIENT_EMAIL?/GOOGLE_DRIVE_PRIVATE_KEY?/GOOGLE_DRIVE_FOLDER_ID?`
- [x] `config/container.ts` — wiring completo (9+2+2 adapters, 9 usecases);
      `EventConflictService` recebe o 5º port (`findConfirmedRankingAgendamentosByCourtUnitAdapter`)
      — fecha a última fonte planejada no kernel
- [x] `applications/routes/routes.ts` — monta `/ranking-agendamentos`
- [x] `container.spec.ts` (+9 chaves) + `routes.spec.ts` (`KNOWN_PREFIXES` + novo `it()` +
      contagem 61→70) atualizados
- [x] `tsc --noEmit` limpo; `eslint` limpo; suíte completa **1003/1003** verde (sem specs novos
      além da atualização dos 2 specs de wiring — cobertura dedicada fica para
      `/speckit-unit-tests`). Lógica central verificada manualmente com script descartável
      (removido): alocação sem extensão (janela exata), create-bulk atômico (sucesso com
      protocolos únicos / rejeição total ao 1º conflito), e attach-proof (1º envio OK, reenvio
      rejeitado com 409).

### Fase 6 — Aplicação e fechamento

- [x] `applications/routes/routes.ts` — monta `/campeonato-agendamentos` e `/ranking-agendamentos`
      (rotas de entidade-mãe `/campeonatos`/`/rankings` não existem mais — ver revisão arquitetural)
- [x] `container.spec.ts` + `routes.spec.ts` — chaves e contagem de rotas atualizadas a cada fase
- [x] ESLint + `tsc --noEmit` limpos
- [x] Confirmado: nenhum import `domain/**`→`infra/**` novo; 1 adapter por verbo/ação nos domínios
      novos

### Fase 7 — `services/beach-center-bff-campeonatos` (bootstrap + CRUD + integração) — ✅ IMPLEMENTADO

> Ver "Atualização 2" no topo do arquivo. Repo estava vazio (só `package.json`); bootstrapado do
> zero usando `beach-center-bff-aulas` como template estrutural. Sem `.spec.ts` (fora do escopo do
> `/speckit-implement` — fica para `/speckit-unit-tests`, próximo passo real desta task).

**Bootstrap**
- [x] `package.json` (+ `axios`), `tsconfig.json`, `eslint.config.mjs`, `jest.config.ts`,
      `nodemon.json`, `index.js`, `.gitignore`
- [x] `src/main.ts`, `src/config/env.ts` (`DB`/`FIREBASE_PROJECT_ID` obrigatórios;
      `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY` opcionais — mesmo padrão de `aulas`),
      `src/config/firebase.ts`
- [x] `src/domain/errors.ts`, `src/domain/usecases/shared/{handle-usecase-error,object-id}.ts`
- [x] Camada de auth idêntica a `aulas`: `domain/models/user.model.ts`,
      `domain/ports/{input,output}/auth*.port.ts`, `domain/usecases/auth/authenticate-request.usecase.ts`,
      `infra/adapters/auth/firebase-verify-id-token.adapter.ts`,
      `infra/adapters/user/find-user-by-firestore-id.adapter.ts`,
      `applications/middlewares/auth.middleware.ts`, `applications/controllers/shared/handle-http-error.ts`

**CRUD `campeonato`/`ranking`/`partida`** (reconstrução do que a Fase 2/3 tinha feito dentro de
`agendamentos` e foi revertido de lá — ver seção "Revisão arquitetural")
- [x] `domain/models/{campeonato,ranking,partida}.model.ts` — `CampeonatoStatus`/`RankingModo`/
      `IPartida` com `id_agendamento?: string` (referência opaca ao bloqueio em `agendamentos`,
      campo novo que não existia na versão revertida — só nasce quando a partida é agendada)
- [x] `domain/usecases/ranking/shared/ranking-modo.validator.ts` (`assertRankingModoConsistency`)
- [x] `domain/usecases/shared/partida-composicao.validator.ts` (`assertPartidaComposicaoValida`)
- [x] `domain/ports/{input,output}/{campeonato,ranking,partida}*.ts`
- [x] `domain/usecases/{campeonato,ranking,partida}/{create,read,update,delete,list}/*.usecase.ts`
- [x] `infra/schemas/{campeonato,ranking,partida}.schema.ts` (`lado_a`/`lado_b` como `Mixed`, mesma
      técnica de `equipment` em `reserva.schema.ts`)
- [x] `infra/adapters/{campeonato,ranking,partida}/**` (5 cada: create/read/update/set-status/list)
- [x] `applications/dto/{campeonato,ranking,partida}.dto.ts` (`partida.dto.ts` com `yup.lazy`
      discriminado por `tipo`, mesmo ajuste de robustez já documentado na Fase 3 original)
- [x] `applications/controllers/{campeonato,ranking,partida}/**` — thin
- [x] `applications/routes/{campeonato,ranking,partida}.route.ts` — ADMIN (`authMiddleware` +
      `requireRole('ADMIN')`)
- [x] `applications/routes/routes.ts` — monta `/campeonatos`, `/rankings`, `/partidas`

**Integração cliente → `agendamentos`** (o lado que faltava desde a revisão arquitetural original —
lá só o contrato/rota interna tinha sido criado, ninguém chamava)
- [x] `domain/ports/output/{campeonato,ranking}-agendamento-client.port.ts` — `ICreateBulk*`,
      `IDeleteBulk*`, `ITrocarDia*` client ports
- [x] `domain/usecases/shared/campeonato-agendamento-conflict.error.ts` — espelha o 409 de
      `agendamentos` (`conflitos[]`), repassado tal-e-qual pra quem chama `/campeonatos/:id/agendar`
      ou `/partidas/:id/trocar-dia`
- [x] `infra/adapters/{campeonato,ranking}-agendamento-client/{create-bulk,delete-bulk,trocar-dia}/*.adapter.ts`
      — `axios` + `x-api-key`, mesmo padrão de `aula-bloqueio-client` (`aulas`)
- [x] `domain/usecases/campeonato/agendar/agendar-campeonato.usecase.ts` — pega partidas `ATIVA`
      sem `id_agendamento`, empurra `quantidade` pro `create-bulk` de `agendamentos`, liga cada
      partida ao `id_agendamento` retornado por índice (idempotente — só agenda o que falta)
- [x] `domain/usecases/ranking/agendar/agendar-ranking.usecase.ts` — mesma lógica, restrito a
      `modo==='PARTIDAS'` e `ATIVO`
- [x] `domain/usecases/partida/trocar-dia/trocar-dia-partida.usecase.ts` — despacha pro client de
      campeonato ou ranking conforme o dono da partida; `usuario_nome` vem de
      `req.databaseUser.name` (autenticado neste MS) e é repassado no corpo pra `agendamentos`
      gravar a auditoria
- [x] `IDeleteCampeonatoUseCase`/`IDeleteRankingUseCase` (usecases) — ganharam a chamada de
      `delete-bulk` (sem `ids`, cancela tudo) ao cancelar o campeonato/ranking inteiro
- [x] `IDeletePartidaUseCase` (usecase) — ganhou a chamada de `delete-bulk` com
      `ids: [id_agendamento]` (libera só o slot daquela partida) ao cancelar uma partida individual
- [x] `applications/dto/{campeonato-agendar,ranking-agendar,partida-trocar-dia}.dto.ts`
- [x] `applications/controllers/campeonato/agendar/*.ts`, `applications/controllers/ranking/agendar/*.ts`,
      `applications/controllers/partida/trocar-dia/*.ts` — capturam
      `CampeonatoAgendamentoConflictError` explicitamente antes do `handleHttpError` genérico
      (mesmo padrão de `create-bulk`/`change-day` em `agendamentos`)
- [x] Rotas: `POST /campeonatos/:id/agendar`, `POST /rankings/:id/agendar`,
      `PATCH /partidas/:id/trocar-dia` (ADMIN)
- [x] `config/container.ts` — wiring completo (bootstrap + 15 usecases/CRUD + 6 adapters de client +
      3 usecases de integração)
- [x] `tsc --noEmit` limpo; `eslint .` limpo a cada etapa (bootstrap → CRUD → integração →
      trocar-dia); `jest` sem specs (0 testes — esperado, cobertura fica pra
      `/speckit-unit-tests`/`/speckit-component-tests`)

**Deliberadamente fora desta task ainda** (não pedido, não implementado):
- `PATCH .../update` de `campeonato_agendamento`/`ranking_agendamento` (só `modalidade`) não tem
  contraparte no MS de campeonatos — não é chamado de lá.
- `GET .../auditoria` (listar trocas de dia) não tem endpoint espelhado no MS de campeonatos — hoje
  só é consultável direto em `agendamentos`.

## Critérios de Aceite (formais)

### Kernel (5 fontes + bloqueio por data)

**AC-1 — `EventConflictService` mescla as 5 fontes sem regressão**
- **Given** um conflito simultâneo de `events_scheduled` (`OUTRO`), `mensalista_plano`,
  `aula_bloqueio` (recorrentes) e `campeonato_agendamento`/`ranking_agendamento` `CONFIRMED`
  (pontuais) na mesma quadra+unidade
- **When** `EventConflictService.getConflicts` roda para uma data/horário que colide com todos
- **Then** os 5 aparecem no resultado, cada um com `source` correto; as 3 fontes recorrentes se
  comportam exatamente como antes da task (specs existentes continuam verdes)

**AC-2 — Bloqueio pontual não vaza pra outras semanas**
- **Given** um `campeonato_agendamento` `CONFIRMED` de `data: "2026-10-18"` (um sábado)
- **When** se verifica conflito/disponibilidade em `"2026-10-25"` (sábado seguinte), mesma
  quadra+horário
- **Then** **não há** conflito — o bloqueio pontual só vale pra `2026-10-18`

**AC-3 — `DRAFT` não bloqueia nada**
- **Given** um `campeonato_agendamento` em `status='DRAFT'`
- **When** se verifica conflito ou disponibilidade de `scheduling` naquele slot
- **Then** o `DRAFT` é invisível — nenhum bloqueio, nenhuma entrada em `exception_conflicts`

### Entidades-mãe

**AC-4 — CRUD de `campeonato`/`ranking`**
- **Given** um `ADMIN` autenticado
- **When** `POST/GET/PATCH/DELETE /api/v1/campeonatos` e `/api/v1/rankings`
- **Then** CRUD funciona; `ranking` com `modo='GERAL'` exige `data_inicio`/`data_fim`
  (`data_inicio < data_fim`); com `modo='PARTIDAS'` os rejeita

### `campeonato_agendamento` — fluxo de conflito de 2 passos

**AC-5 — `verificar` não persiste nada**
- **Given** um campeonato e um conjunto de quadras/datas/partidas sem nenhum conflito existente
- **When** `POST /campeonatos/:id/agendamentos/verificar`
- **Then** retorna os candidatos alocados (1 slot por partida) e `conflitos: []`; nenhum
  `campeonato_agendamento` é criado no banco

**AC-6 — `confirmar` com `cancelar_conflitos=false` gera `DRAFT` e não mexe em nada**
- **Given** o mesmo cenário de AC-5, mas agora com 1 conflito real (um `mensalista_plano`
  ocupando 1 dos slots)
- **When** `POST .../confirmar` com `cancelar_conflitos: false`
- **Then** todos os candidatos (incl. o conflitante) são criados como `campeonato_agendamento`
  `DRAFT`; o `mensalista_plano` conflitante **continua intacto e bloqueando normalmente**; nenhum
  `scheduling` muda de `available`

**AC-7 — `confirmar` com `cancelar_conflitos=true` cancela o conflito e confirma o campeonato**
- **Given** o mesmo cenário de AC-6
- **When** `POST .../confirmar` com `cancelar_conflitos: true`
- **Then** o `mensalista_plano` conflitante é cancelado (`status='CANCELLED'`, quadra liberada
  onde aplicável) e todos os candidatos nascem `campeonato_agendamento` `CONFIRMED`, bloqueando os
  `scheduling`s correspondentes (`available=false`)

**AC-8 — Falta de quadra/data suficiente é rejeitado**
- **Given** 5 partidas e só 1 quadra + 1 data com espaço pra 2 slots antes do fechamento da
  unidade
- **When** `POST .../verificar`
- **Then** `400` — não aloca parcialmente, pede mais quadras/datas

**AC-9 — Bloqueio exato por quadra+unidade**
- **Given** um `campeonato_agendamento` `CONFIRMED` na quadra 1 da unidade A
- **When** se consulta disponibilidade da quadra 2 da unidade A, ou da quadra 1 de uma unidade B
- **Then** ambas continuam livres — o bloqueio nunca extrapola `court`+`unit` exatos

**AC-10 — Bloqueio do campeonato estende até o fechamento quando `hora_fim` é tardia**
- **Given** um `campeonato_agendamento` com `hora_inicio: "20:00"`, `hora_fim: "23:30"`, numa
  unidade cujo fechamento é `22:00`
- **When** o bloqueio é calculado/aplicado
- **Then** a janela efetiva vai até `22:00` (fechamento), não `23:30`; se `hora_fim` fosse
  `"21:00"` (antes do fechamento), a janela efetiva seria até `21:00`

**AC-11 — Composição de partida — mismatch é rejeitado**
- **Given** um candidato com `lado_a.tipo='DUPLA'` e `lado_b.tipo='EQUIPE'`
- **When** `POST .../verificar` ou `.../confirmar`, ou `POST /rankings/:id/agendamentos`
- **Then** `400` antes de qualquer alocação/persistência — dupla só joga contra dupla, solo só
  contra solo, equipe só contra equipe

### `campeonato_agendamento` — troca de dia, delete, auditoria

**AC-12 — Trocar o dia sem conflito**
- **Given** um `campeonato_agendamento` `CONFIRMED`
- **When** `PATCH .../:id/trocar-dia` com `dia_novo` livre + `motivo`
- **Then** o slot antigo é liberado (`scheduling.available=true` se elegível), o novo é bloqueado,
  o registro passa a ter `data=dia_novo`, e uma auditoria é criada com `usuario_nome` (do token),
  `motivo`, `dia_anterior`, `dia_novo`

**AC-13 — Trocar o dia com conflito exige confirmação**
- **Given** o `dia_novo` já tem um conflito (ex.: `aula_bloqueio`)
- **When** `PATCH .../trocar-dia` **sem** `cancelar_conflitos: true`
- **Then** retorna os conflitos sem mover nada; com `cancelar_conflitos: true`, cancela o
  conflitante e move normalmente (AC-12)

**AC-14 — Delete individual e em massa**
- **Given** um `campeonato_agendamento` `CONFIRMED` e outros 2 do mesmo `id_campeonato`
- **When** `PATCH .../:id/delete` (1) e depois `PATCH .../delete-bulk` sem `ids` (resto)
- **Then** todos ficam `CANCELLED` e os `scheduling`s elegíveis voltam a `available=true`

**AC-15 — Listar auditoria**
- **Given** 2 trocas de dia já feitas
- **When** `GET .../agendamentos/auditoria`
- **Then** retorna as 2 entradas, cada uma com `usuario_nome`/`motivo`/`dia_anterior`/`dia_novo`

### `ranking_agendamento` — conflito simples, protocolo, pagamento

**AC-16 — Criar ranking_agendamento com conflito é rejeitado (409, sem fluxo especial)**
- **Given** um slot já ocupado por qualquer bloqueador CONFIRMED
- **When** `POST /rankings/:id/agendamentos`
- **Then** `409`/`400` — RANKING **não** tem prioridade, rejeita direto (diferente do campeonato)

**AC-17 — Bloqueio do ranking é a janela exata**
- **Given** um `ranking_agendamento` `hora_inicio:"20:00"` `hora_fim:"21:00"` numa unidade que
  fecha `22:00`
- **When** o bloqueio é aplicado
- **Then** só `20:00`–`21:00` fica indisponível; `21:00`–`22:00` continua livre (ao contrário do
  campeonato, AC-10)

**AC-18 — `numero_protocolo` é único globalmente**
- **Given** um `numero_protocolo` já usado por uma `reserva` existente
- **When** um `ranking_agendamento` é criado e por azar sorteia o mesmo número
- **Then** o sistema gera outro número antes de persistir — nunca há colisão entre as 2 coleções

**AC-19 — Anexar comprovante muda o status e é público**
- **Given** um `ranking_agendamento` `status='pending'` e seu `numero_protocolo`
- **When** `PATCH /ranking-agendamentos/protocolo/:numero_protocolo/comprovante` (sem token
  Firebase) com um PDF/imagem
- **Then** `200`, comprovante salvo (Google Drive se configurado, senão base64), `status` vira
  `waiting_approve`

**AC-20 — Reenviar comprovante depois de já enviado é rejeitado**
- **Given** um `ranking_agendamento` já em `waiting_approve` (ou `approved`/`rejected`)
- **When** o mesmo endpoint de anexar é chamado de novo
- **Then** `409` — não substitui o comprovante já enviado

**AC-21 — "Ranking geral" não gera agendamento nem bloqueio**
- **Given** um `IRanking` com `modo='GERAL'`
- **When** ele é criado
- **Then** nenhum `ranking_agendamento` é gerado e nenhum `scheduling` é afetado — reservas nesse
  período seguem o fluxo comum de `POST /agendamentos`/reservas

### Autorização

**AC-22 — Tudo ADMIN, exceto o comprovante**
- **Given** requisições sem token Firebase válido
- **When** qualquer endpoint de `/campeonatos`, `/rankings`, `/campeonatos/:id/agendamentos`,
  `/rankings/:id/agendamentos`
- **Then** `401`/`403`; **exceto** `PATCH /ranking-agendamentos/protocolo/:numero/comprovante`,
  que funciona sem token (D12)

### Qualidade (Princípio III — cobrado em `/speckit-unit-tests`)

**AC-23 — Isolamento de camadas e specs do kernel**
- **Given** o repo após a task
- **When** ESLint + Jest com cobertura rodam
- **Then** nenhum import `domain/**`→`infra/**` novo; specs de `event-conflict.service`,
  `event-scheduling-impact.service`, `list-day-schedulings` cobrem as 5 fontes sem regressão;
  cobertura global ≥ 80%, incluindo o `slot-allocator` (casos de borda do algoritmo de alocação)

### `beach-center-bff-campeonatos` — CRUD e integração cliente (Fase 7)

**AC-24 — CRUD de `campeonato`/`ranking`/`partida` no MS correto**
- **Given** um `ADMIN` autenticado no MS de campeonatos
- **When** `POST/GET/PATCH /campeonatos`, `/rankings`, `/partidas`
- **Then** CRUD funciona; `partida` exige exatamente um de `id_campeonato`/`id_ranking` (XOR),
  valida que o dono está `ATIVO` (e, se ranking, `modo==='PARTIDAS'`), e rejeita composição
  `lado_a.tipo !== lado_b.tipo`

**AC-25 — Agendar um campeonato empurra o bloqueio pra `agendamentos` e liga as partidas**
- **Given** um campeonato `ATIVO` com N partidas `ATIVA` sem `id_agendamento`
- **When** `POST /campeonatos/:id/agendar` com quadras/datas/horário válidos
- **Then** `agendamentos` recebe 1 chamada de `create-bulk` com `quantidade: N`; cada partida
  retornada é ligada ao `id_agendamento` correspondente, na mesma ordem; chamar de novo sem novas
  partidas não reagenda as já ligadas (idempotente)

**AC-26 — Conflito ao agendar campeonato propaga a decisão ponta a ponta**
- **Given** o cenário de AC-25, mas com 1 slot em conflito
- **When** `POST /campeonatos/:id/agendar` sem `cancelar_conflitos`
- **Then** `409` com a lista de `conflitos` (mesmo formato de `agendamentos`); resubmeter com
  `cancelar_conflitos: true`/`false` segue o mesmo comportamento documentado em AC-6/AC-7

**AC-27 — Cancelar libera o bloqueio no serviço certo**
- **Given** um campeonato/ranking com partidas já agendadas, e uma partida específica também já
  agendada
- **When** (a) `PATCH /campeonatos/:id/delete` (ou `/rankings/:id/delete`), e (b)
  `PATCH /partidas/:id/delete` de uma partida à parte
- **Then** (a) `agendamentos` recebe `delete-bulk` sem `ids` (cancela todos os agendamentos daquele
  `id_campeonato`/`id_ranking`); (b) `agendamentos` recebe `delete-bulk` com
  `ids: [id_agendamento]` (cancela só o daquela partida, sem afetar as demais)

**AC-28 — Trocar o dia de uma partida aciona a auditoria em `agendamentos`**
- **Given** uma partida já agendada (`id_agendamento` presente)
- **When** `PATCH /partidas/:id/trocar-dia` com `dia_novo`/`motivo`
- **Then** a chamada é repassada pra `trocar-dia` de `campeonato-agendamentos`/
  `ranking-agendamentos` conforme o dono, com `usuario_nome` do `req.databaseUser` autenticado
  neste MS; conflito no dia novo (só campeonato) propaga `409`+`conflitos` como em AC-26; partida
  sem `id_agendamento` é rejeitada com `400` ("ainda não foi agendada")

## Riscos e observações

- **Maior task do ecossistema até agora** — 2 domínios de agendamento (cada um com fluxo próprio
  de conflito) + 2 entidades-mãe + kernel estendido para bloqueio pontual + protocolo/comprovante
  autocontidos. Considerar dividir o `/speckit-implement` em passadas (kernel → campeonato →
  ranking → aplicação/rotas), como a task 004 fez.
- **`slot-allocator` é lógica nova, não um CRUD replicado** — é o único pedaço desta task sem um
  precedente direto nas tasks 002-004. Merece atenção redobrada em specs (datas insuficientes,
  quadras insuficientes, partida "sobrando", fechamento de unidade diferente por unidade).
- **`conflict-resolution.service` precisa tocar em usecases de 4-5 domínios diferentes**
  (mensalista, aula, events_scheduled, ranking, reserva/scheduling) só para cancelar — é um ponto
  de acoplamento novo no kernel; isolar atrás de ports por domínio (já listados no mapa) para não
  criar import direto entre domínios de usecase.
- **Duplicação deliberada do Google Drive adapter** — mesma técnica de
  `beach-center-bff-pagamentos`, copiada, não importada (decisão do usuário para não integrar
  serviços nesta task). Ficam 2 cópias quase idênticas no ecossistema — candidato a
  extrair/compartilhar numa task futura, mas fora de escopo agora.
- **`IReserveStatus` compartilhado ganhando `'waiting_approve'`** — mudança de tipo aditiva, mas
  em um model usado por muitos usecases de reserva comum; conferir no `/speckit-implement` que
  nenhum `switch`/validação exaustiva de status (`oneOf` em DTO, por exemplo) precise de ajuste
  pra não rejeitar o novo valor onde ele puder aparecer (mesmo que só `ranking_agendamento` o
  produza agora).
- **Algoritmo de alocação determinístico é pré-requisito do fluxo `verificar`→`confirmar`** — as
  2 chamadas **têm** que gerar os mesmos candidatos a partir do mesmo payload (sem estado no
  servidor entre elas); qualquer não-determinismo (ex.: ordenação de `Promise.all` afetando
  resultado) quebra o contrato.
- **`beach-center-app`** — nenhuma tela para os fluxos acima (verificar/confirmar,
  trocar-dia com conflito, anexar comprovante) existe hoje; front é tarefa futura, mas os
  contratos de API acima devem ser documentados com clareza extra no `/speckit-documentation` por
  serem novos padrões (dry-run + confirm) no ecossistema.

## Próximo passo

~~`/speckit-implement`~~ — **concluído** (Fases 1, 4, 5, 6 em `beach-center-bff-agendamentos`;
Fase 7 em `beach-center-bff-campeonatos`; Fases 2/3 revertidas de propósito, ver Revisão
arquitetural). Nenhum commit/push foi feito em nenhum dos dois repos.

Falta, nesta ordem:
1. ~~`/speckit-unit-tests`~~ — **concluído.** `beach-center-bff-agendamentos`: 275 suites / 1289
   testes (0 falhas), cobertura global 98.85% stmts / 92.8% branches / 98.85% funcs / 98.94% lines
   — inclui specs novos para `slot-allocator`, `conflict-resolution.service`, extensão de
   `event-conflict.service`/`event-scheduling-impact.service` (5 fontes + `specific_date`, AC-1/
   AC-2/AC-3, sem regressão nas 3 fontes antigas), e os domínios inteiros de
   `campeonato-agendamento`/`ranking-agendamento` (usecases, adapters, schemas, DTOs, controllers,
   auditoria, protocolo, Google Drive). `beach-center-bff-campeonatos`: 83 suites / 329 testes (0
   falhas), cobertura 98.63% stmts / 88.4% branches / 100% funcs / 98.58% lines — CRUD completo,
   `partida-composicao.validator`, `agendar-*`/`trocar-dia-partida` com os clients HTTP mockados.
   `tsc --noEmit` e `eslint .` limpos nos dois repos. **Bug real encontrado e corrigido pelos
   testes**: `updatePartidaDTO` (campeonatos) rejeitava updates parciais válidos — `.optional()`
   encadeado num `yup.lazy` não impedia a resolução do schema interno quando o campo estava
   ausente; corrigido com um guard explícito de `value === undefined` antes do switch (ver
   `applications/dto/partida.dto.ts`).
2. ~~`/speckit-component-tests`~~ — **N/A confirmado (2026-09-09).** A Constituição (Princípio III)
   restringe `Cypress` + `Cucumber` a testes de componente/E2E de **Frontend (React)**. Esta task
   toca só `beach-center-bff-agendamentos` e `beach-center-bff-campeonatos` (Node/TS, backend
   puro) — `beach-center-app` está fora de escopo e não tem nenhuma tela para os fluxos novos
   (verificar/confirmar, trocar-dia com conflito, anexar comprovante). Nenhum dos dois repos tem
   Cypress configurado nem arquivos `.feature`. Todos os 28 Critérios de Aceite (AC-1..AC-28) são
   de nível API/domínio e já estão cobertos por `/speckit-unit-tests` (item 1). Nenhum arquivo
   `.feature`/step gerado; nenhum commit.
3. ~~`/speckit-validate`~~ — **concluído (2026-09-09).** Revisão arquivo-a-arquivo dos 169 arquivos
   de `beach-center-bff-agendamentos` + 193 de `beach-center-bff-campeonatos` (produção + specs).
   Sem violação de camada (nenhum import `domain/**`→`infra/**`), aderência ao plano/ACs
   confirmada. **1 correção aplicada** (`beach-center-bff-agendamentos/src/config/container.ts` +
   `conflict-resolution.service.ts`): o `ConflictResolutionService` não estava recebendo o
   `deleteRankingAgendamentoUsecase` — um conflito `CAMPEONATO×RANKING` resolvido com
   `cancelar_conflitos: true` lançaria `"Cancelamento de conflito RANKING nao configurado"` em
   runtime (AC-7/AC-26 para fonte RANKING). Fase 5 previa "passa a injetar" mas o wiring ficou de
   fora. Corrigido: `deleteRankingAgendamentoUsecase` movido para antes do
   `ConflictResolutionService` e passado como 6º argumento; comentário obsoleto da interface local
   atualizado. `tsc --noEmit` + `eslint .` + Jest limpos nos dois repos após a correção
   (agendamentos 275/1289, campeonatos 83/329). Alterações staged nos dois repos, sem commit.
4. ~~`/speckit-test`~~ — **concluído (2026-09-09).** `tasks/005-agendamento-campeonato-ranking/exploratory-tests.md`
   gerado: 12 caminhos felizes (HP-01..HP-12), 25 fluxos de exceção/autorização (EX-01..EX-25),
   15 edge cases (ED-01..ED-15), 11 itens de regressão (RG-01..RG-11). Todos os 28 AC rastreados a
   ≥ 1 cenário; 6 cenários marcados como "só manual" (integração/infra/limites). Nenhum código
   alterado.
5. `/speckit-complete` → `/speckit-documentation` (esta última precisa documentar os dois serviços
   e o contrato de integração entre eles — algo novo no ecossistema).
