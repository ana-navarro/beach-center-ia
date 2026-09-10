# Task 005 — Agendamento de Campeonato e Ranking (continuação do desmonte de `events_scheduled`)

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> **Contém correções à proposta inicial do usuário** (o usuário pediu ajuda com o que estivesse errado) e uma
> lista de **perguntas em aberto** que precisam de resposta antes do `/speckit-plan`.
>
> **Atualização — decisões já confirmadas pelo usuário** (ver seção "Decisões confirmadas" abaixo):
> recorrência por **data específica** (não dia-da-semana); **participantes/partidas ENTRAM no
> escopo** desta task (reverte a recomendação inicial C-3) com `lado_a`/`lado_b` + `tipo`
> discriminante; **duas coleções separadas** para campeonato e ranking; regra de **bloqueio
> diferenciada** entre campeonato e ranking (fechamento dinâmico da unidade); **ambos** têm
> **entidade-mãe leve** (`campeonato`/`ranking`) dona dos agendamentos; **RANKING usa a mesma
> auditoria de troca de dia do CAMPEONATO**; existe um segundo modo de ranking — **"ranking
> geral"** (evento por período, sem agendamentos gerados — reserva de quadra é a reserva comum de
> usuário) — que entra só como **cadastro leve, sem lógica de bloqueio**.
>
> **Atualização 2 (2026-09-09, durante o `/speckit-implement`) — `services/beach-center-bff-campeonatos`
> DEIXOU de ficar fora de escopo.** No meio da implementação, o usuário pediu que `campeonato`/
> `ranking`/`partida` seguissem o **mesmo padrão de integração da aula** (task 004): dono do domínio
> de negócio (agora o MS de campeonatos, não mais `agendamentos`) empurra o bloqueio de quadra via
> rota **interna** (`x-api-key`) — `agendamentos` nunca chama o MS de campeonatos de volta. Isso
> reverte a decisão 6/C-5(a) abaixo (que dizia "sem MS de campeonatos, tudo dentro de
> `agendamentos`") e o item "Serviço(s) alvo" logo adiante. Detalhes completos — o que foi revertido
> de `agendamentos`, o que foi construído do zero em `beach-center-bff-campeonatos`, e a nova
> integração (`agendar`, `trocar-dia`, cancelamento liberando bloqueio) — estão no `plan.md`, seção
> "⚠️ Revisão arquitetural" e Fase 7. Este `context.md` fica como registro histórico das decisões
> originais; não foi reescrito por baixo para não perder o histórico do "porquê".

## Título

**Separar "campeonato" e "ranking" da tabela genérica de exceções (`events_scheduled`)**, criando
domínios dedicados dentro de `beach-center-bff-agendamentos` — mesma linha das tasks 004
(`mensalista_plano`, `aula_bloqueio`).

Objetivo maior (do usuário): *"tirar cada vez mais as exceções"* — desmontar `events_scheduled` em
modelos por domínio. Ordem já feita: **task 004** tirou `MENSALISTA` → `mensalista_plano` e
`AULA_BEACH_TENIS`/`AULA_VOLEI` → `aula_bloqueio`. **Esta task** tira campeonato/ranking. Ao final,
`events_scheduled` fica **só com `OUTRO`** (a exceção verdadeiramente pontual — o último resquício).

## Descrição (a partir do input do usuário)

- **Sistema de Campeonato *dentro de* Agendamento** — gera agendamentos de quadra para um campeonato,
  **sem envolver o MS de campeonatos** (`services/beach-center-bff-campeonatos` — hoje repo vazio,
  só `package.json`).
- Quando o usuário cria um campeonato e gera as partidas, o sistema lança os agendamentos
  (exceções) e **não permite reservar a quadra no mesmo dia de um campeonato**.
- Endpoints previstos:
  - CRUD comum: `create`, `read`, `update`, `delete`, `list`.
  - **`agendamento em massa`** (criar N agendamentos de uma vez).
  - **`deleta em massa`**.
  - **`atualizar somente o dia do campeonato`** — com **auditoria**.
- **Auditoria de troca de dia**: ao mover o dia, o usuário escolhe um **dia novo no calendário** e
  informa um **motivo**; grava-se `usuario_nome`, `motivo`, `dia_anterior`, `dia_novo` (+ timestamp).
  Mover o dia também **re-libera o dia antigo** e bloqueia o novo nos `scheduling`s.
- Regra de composição de partida: *"não pode ter dupla contra equipe ou solo; sempre dupla×dupla,
  equipe×equipe, solo×solo"*.

### Interfaces fornecidas pelo usuário (ponto de partida — **contém pontos a corrigir**)

```ts
interface IDupla { participante_a: string; tel_participante_a: string; participante_b: string; tel_participante_b: string; }
interface ISolo  { participante: string; tel_participante: string; }
interface IEquipe { participantes: string[]; tel_capitao: string; }
interface IPartida { participantes: IDupla | ISolo | IEquipe; }

interface IAgendamentoCampeonatoCommon {
  dia: string[];        // "dia de semanas [segunda..domingo]"  ← ver correção #1
  horaInicio: Date;
  horaFim: Date;
  quadra: string;
  modalidade: string;
}
interface ICampeonatoAgendamentoPartidaCommon extends IAgendamentoCampeonatoCommon { tipo: 'CAMPEONATO'; id_campeonato: string; }
interface IRankingAgendamentoPartidaCommon    extends IAgendamentoCampeonatoCommon { tipo: 'RANKING';    id_ranking: string; }
```

## Decisões confirmadas (respostas do usuário — 2026-09-09)

1. **Recorrência (C-2): data específica.** Cada agendamento de campeonato/ranking tem `data`
   (`"YYYY-MM-DD"`) própria — não `day_of_week`. Um campeonato de vários dias/partidas gera **um
   registro por partida/data** (consistente com "agendamento em massa"). Confirma C-2.

2. **Escopo (C-3 REVERTIDA): participantes/partidas ENTRAM no escopo.** Diferente da recomendação
   inicial, o usuário confirmou que quer guardar `IDupla`/`ISolo`/`IEquipe` e a partida dentro do
   agendamento de `agendamentos` (não só dados de quadra/horário). Ver **C-3-bis** abaixo — a
   interface `IPartida` fornecida precisa de correção estrutural para isso funcionar.

3. **Coleções: duas separadas.** `campeonato_agendamento` (nome a fechar) e `ranking_agendamento`
   — mesmo padrão de `mensalista_plano` + `aula_bloqueio` da task 004 (portas/adapters/usecases
   duplicados por domínio, não uma coleção com campo `tipo`).

4. **Bloqueio de quadra — regra DIFERENTE por tipo** (resposta literal do usuário): *"Campeonato
   bloqueia a partir da horaInicio, se a hora fim for antes das 22 horas aí vale olhar a hora fim,
   Ranking hora inicio e hora fim"*. Interpretação proposta (**confirmar no `/speckit-plan`**):
   - **CAMPEONATO**: bloqueia a partir de `hora_inicio`. Se `hora_fim` for **antes do horário de
     fechamento da unidade**, bloqueia só até `hora_fim`. Se `hora_fim` for **igual/depois do
     fechamento** (o "22 horas" do usuário é o fechamento da unidade principal — ver
     `SchedulingWindowValidator.resolveOperatingHours`, hoje `22:00` para a unidade
     `6a440a931094fad2f585011b` e `23:00` para as demais), bloqueia até o **fechamento real da
     unidade** (não um "22h" fixo hard-coded — reaproveitar `resolveOperatingHours(unit).end_hour`
     para não quebrar em unidades com fechamento diferente). *(Confirmar esta leitura — pergunta
     nova #A abaixo.)*
   - **RANKING**: bloqueia exatamente a janela `hora_inicio`–`hora_fim` — igual ao padrão simples
     de `aula_bloqueio`/`mensalista_plano`, sem extensão até o fechamento.
   - Isso já responde parte de C-6 (RANKING ≠ CAMPEONATO também na regra de bloqueio, não só no id).
   - **Confirmado (2026-09-09):** a leitura acima está correta — antes do fechamento usa `hora_fim`,
     senão usa o fechamento real da unidade (dinâmico, não fixo).

5. **Formato `lado_a`/`lado_b` (C-3-bis, pergunta B): aceito.** `IPartida` vira
   `{ lado_a: IDupla | ISolo | IEquipe; lado_b: IDupla | ISolo | IEquipe }`, cada lado com `tipo:
   'DUPLA' | 'SOLO' | 'EQUIPE'` explícito. Regra de composição = `lado_a.tipo === lado_b.tipo`.

6. **Entidade-mãe (C-5, pergunta #3): SIM, para os dois domínios.** `agendamentos` cria um
   documento leve `campeonato` e um documento leve `ranking` (nome, modalidade, etc.), donos dos
   respectivos agendamentos/partidas — mesmo padrão de `mensalista` → `mensalista_plano` (task
   004). Motivo dado pelo usuário: viabilizar `agendamentos` como **ponto de referência mais
   completo** do projeto no futuro (ex.: outros serviços podem querer consultar
   `campeonato`/`ranking` por aqui). *Implicação para o `/speckit-plan`:* isso são 2 novas
   entidades-mãe + 2 novas entidades de agendamento/partida = 4 novos modelos/coleções no total
   (mais a de auditoria).

7. **RANKING — natureza e comportamento (pergunta #5), com 2 sub-decisões:**
   - **7a. Troca de dia de uma partida de ranking usa A MESMA auditoria do campeonato**
     (motivo + `usuario_nome` + `dia_anterior`/`dia_novo`) — não é um update comum sem rastro.
     Ambos os domínios (`campeonato_agendamento` e `ranking_agendamento`) têm o endpoint
     dedicado de troca-de-dia com auditoria; a única assimetria entre os dois é a regra de
     **bloqueio de horário** (item 4), não a auditoria.
   - **7b. Existe um segundo "tipo" de ranking — "ranking geral"**: um evento que dura um
     **período** (data_início/data_fim), **sem gerar nenhum agendamento/exceção** — durante esse
     período, o usuário reserva a quadra **como reserva comum** (fluxo normal de `scheduling`,
     fora do sistema de exceções desta task). **Decisão confirmada:** esta task cria **só o
     cadastro leve** desse "ranking geral" (nome, `data_inicio`, `data_fim`, modalidade) — **sem
     nenhuma lógica de bloqueio/conflito associada a ele** nesta task. Isso é distinto da
     "partida marcada de ranking" (item 7a), que **é** um agendamento com bloqueio, igual ao
     campeonato.
   - *(Consequência de modelagem: a entidade-mãe `ranking` do item 6 provavelmente precisa
     suportar os dois modos — com partidas agendadas, ou "geral" sem nenhuma. Decisão de schema
     exata fica para o `/speckit-plan`.)*

8. **Pagamento/comprovante — SÓ para RANKING (partida marcada), não campeonato** (respostas
   2026-09-09): quando o usuário anexa o comprovante de pagamento de uma partida de ranking, o
   agendamento passa para um status **`waiting_approve`** (novo valor no enum compartilhado
   `IReserveStatus` de `beach-center-bff-agendamentos` — hoje `pending|approved|rejected|
   cancelled` — mas **sem alterar o comportamento atual da reserva comum**; só `ranking_agendamento`
   efetivamente usa/seta esse valor nesta task). Além disso, `ranking_agendamento` ganha um
   **número de protocolo único** (mesmo padrão de `IReserve.number`, gerado com
   `generateNumericId()`), usado para localizar/gerenciar o agendamento em operações como
   "trocar o dia". **Confirmado: sem integrar com `beach-center-bff-pagamentos` nesta task** — o
   fluxo de comprovante fica autocontido em `agendamentos` (sem chamar
   `submit-manual-payment`/`review-manual-payment` do MS de pagamentos). CAMPEONATO **não** ganha
   esse status/protocolo de pagamento — ele não envolve pagamento de usuário final.

11. **Quadra do CAMPEONATO — 1 registro = 1 quadra (confirmado, resolve a tensão do item 7).** A
    "regra de várias quadras" não é um campo array num registro — é o **agendamento em massa**
    criando **vários registros, um por quadra/data/horário**, cada um ainda com **uma** partida
    (`lado_a`/`lado_b`) e **uma** `quadra`. Igual ao padrão de `ranking_agendamento`.

12. **Bloqueio exato por quadra+unidade (C-4/#8, confirmado com exemplo do usuário):** um
    agendamento de campeonato na quadra 1 da unidade A bloqueia **só** a quadra 1 da unidade A. As
    demais quadras da unidade A (se houver, ex.: quadras 2–7) ficam livres para
    agendamento/mensalista/aula, e a quadra 1 de uma unidade B (diferente) também fica livre —
    matching sempre por **`court` + `unit`** exatos, nunca por unidade inteira.

13. **CAMPEONATO tem prioridade máxima — com fluxo de conflito em 2 passos (novo, importante).**
    Ao criar (unitário ou em massa) um agendamento de campeonato, o sistema **não valida contra**
    `mensalista_plano`/`aula_bloqueio`/`events_scheduled` `OUTRO`/reserva comum/`ranking_agendamento`
    automaticamente — ele **detecta os conflitos existentes** naquela quadra+data+horário e
    devolve ao admin **a lista desses conflitos**, pedindo confirmação:
    - Se o admin confirmar **"cancelar"** → todos os agendamentos/reservas conflitantes daquela
      quadra+data+horário são **cancelados/liberados**, e o campeonato é criado **confirmado**,
      bloqueando o slot normalmente.
    - Se o admin confirmar **"não cancelar"** → **nada é cancelado**, e o agendamento de campeonato
      é criado como **`DRAFT`** (rascunho) — **não bloqueia o slot** (as reservas existentes
      continuam intactas) — serve só de anotação para o admin escolher outra data depois (via
      "trocar o dia", presumivelmente).
    - *(Implicação de design para o `/speckit-plan`: `create`/`create-bulk` do campeonato precisam
      de um fluxo de 2 etapas — "verificar conflitos" (dry-run, retorna a lista) e "confirmar"
      (com a decisão `cancelar: true/false`) — e um novo status `DRAFT` no
      `campeonato_agendamento`, que fica **fora** do `EventConflictService`/bloqueio enquanto
      nesse estado. Detalhe exato do contrato de API é do plano, não desta task.)*
    - Isso também é a resposta final do item #9: a geração em massa **"gera sozinho"** (o endpoint
      recebe `id_campeonato` + parâmetros e monta os agendamentos), e por ser prioridade máxima,
      **ignora** os outros tipos de bloqueio ao gerar — o conflito só aparece nesse fluxo de
      confirmação acima, não como uma rejeição 409 simples (diferente do que eu tinha recomendado
      antes para #12 — **revisado**: `RANKING`/`mensalista_plano`/`aula_bloqueio`/reserva comum
      continuam rejeitando 409 normalmente entre si; só `CAMPEONATO` tem esse fluxo especial de
      prioridade/conflito.

## Serviço(s) alvo (Princípio I)

| Serviço | Papel | Justificativa |
|---|---|---|
| **`services/beach-center-bff-agendamentos`** | Dono das coleções de agendamento (`campeonato_agendamento`/`ranking_agendamento`/auditorias), do kernel de conflito/bloqueio, e da redução de `events_scheduled`. | O bloqueio de quadra (dias/horário/conflito/`scheduling.available`) é regra de `agendamentos` — já vive lá como `events_scheduled`. |
| **`services/beach-center-bff-campeonatos`** | **Passou a ser tocado (Atualização 2 acima).** Dono de `campeonato`/`ranking`/`partida` (entidade-mãe + composição), e de quem chama `agendamentos` via rota interna. | Pivô arquitetural pedido pelo usuário no meio do `/speckit-implement` — mesmo padrão da aula (task 004): quem é dono do dado de negócio empurra o bloqueio pra `agendamentos`, nunca o contrário. Repo estava vazio (só `package.json`) antes desta task; foi bootstrapado do zero (ver `plan.md`, Fase 7). |
| ~~`beach-center-app`~~ | **NÃO tocado.** | Front fora de escopo (consistente com tasks 002/003/004). A task entrega só a API. |

## Correções à proposta inicial (o usuário pediu ajuda)

### C-1 — "campeonato/ranking já estava em `events_scheduled`" → **não estava**

O enum de `events_scheduled` **hoje** é só `'AULA_BEACH_TENIS' | 'MENSALISTA' | 'AULA_VOLEI' | 'OUTRO'`
(schema + model). **Nunca houve `CAMPEONATO` nem `RANKING`.** Um bloqueio de campeonato hoje seria
criado como `event_type: 'OUTRO'`, sem forma de distinguir depois. **Consequência:**
- **Não há migração** de dados legados nesta task (diferente da task 004). Se existirem `OUTRO`
  que são "na verdade" campeonatos, eles não podem ser convertidos automaticamente — ficam como
  `OUTRO` (o admin recria como campeonato se quiser). *(Confirmar — pergunta #12.)*
- Não é preciso "remover CAMPEONATO do `events_scheduled`" — ele nunca esteve lá.

### C-2 — `dia: string[]` como **dias da semana** contradiz o resto da descrição — **RESOLVIDO**

`mensalista_plano` e `aula_bloqueio` são **recorrências semanais** ("toda segunda e quarta"). Um
**campeonato acontece em data(s) específica(s)** ("neste sábado 18/10", ou "18–19/10"). Os próprios
requisitos do usuário confirmam isso:
- *"não permitir reservar a quadra **no mesmo dia** que um campeonato"* — "dia" = **data**.
- *"selecionar um **dia novo no calendário**"* / *"atualizar somente o **dia** do campeonato"* — data pontual.
- Auditoria com `dia_anterior` / `dia_novo` — datas concretas, não "de terça para quinta".

**Resolvido:** cada agendamento de campeonato/ranking usa **uma data específica**
(`data: string "YYYY-MM-DD"`) — **não** `day_of_week`. Um campeonato com várias partidas/dias gera
**um registro por partida** (cada um com sua `data`), lançados via "agendamento em massa". O
horário (`hora_inicio`/`hora_fim`) continua `"HH:MM"` (string, como `events_scheduled`).

### C-3 — `IPartida`/`IDupla`/`ISolo`/`IEquipe` + regra "dupla×dupla" — **REVERTIDO: fica no escopo**

*(Recomendação original: deixar isso fora de `agendamentos`, por ser gestão de campeonato. O
usuário respondeu que quer **incluir** participantes/partidas nesta task mesmo assim — decisão
dele, registrada. Mantém-se o registro da recomendação original para o histórico, mas a decisão
vigente é a de baixo.)*

**Decisão vigente:** o agendamento de campeonato/ranking guarda, além dos dados de quadra/data/
horário, os dados da **partida** (`IDupla`/`ISolo`/`IEquipe`) e valida a regra de composição
(dupla×dupla, equipe×equipe, solo×solo) **dentro de `beach-center-bff-agendamentos`**, sem chamar
o MS de campeonatos (que segue vazio/fora de escopo — só o `id_campeonato`/`id_ranking` é uma
referência externa opaca).

#### C-3-bis — a interface `IPartida` fornecida não sustenta a regra "dupla×dupla" (correção necessária)

```ts
// Como fornecida:
interface IPartida { participantes: IDupla | ISolo | IEquipe; }
```

Isso descreve **um lado só**. Uma partida tem **dois lados competindo**, e a regra "não pode ter
dupla contra equipe" exige comparar os dois. **Correção proposta:**

```ts
type TipoParticipante = 'DUPLA' | 'SOLO' | 'EQUIPE';

interface IDupla  { tipo: 'DUPLA';  participante_a: string; tel_participante_a: string; participante_b: string; tel_participante_b: string; }
interface ISolo   { tipo: 'SOLO';   participante: string; tel_participante: string; }
interface IEquipe { tipo: 'EQUIPE'; participantes: string[]; tel_capitao: string; }

interface IPartida {
  lado_a: IDupla | ISolo | IEquipe;
  lado_b: IDupla | ISolo | IEquipe;
}
```

- Cada lado ganha um discriminante explícito `tipo: 'DUPLA' | 'SOLO' | 'EQUIPE'` — sem isso, a
  validação yup teria que adivinhar o tipo pelas chaves presentes (frágil: `IEquipe` com 2
  participantes pode ser confundido com outra forma). Com `tipo` explícito, a regra vira
  `lado_a.tipo === lado_b.tipo` — trivial de validar num DTO.
- *(Confirmar — pergunta nova #B: aceitar esse formato `lado_a`/`lado_b` + `tipo` explícito? É a
  única forma prática de validar a regra de composição de forma confiável.)*

### C-4 — "não permitir reservar no mesmo **dia**" vs `hora_inicio`/`hora_fim` — **RESOLVIDO (regra assimétrica)**

**Resolvido — regra diferente por tipo** (ver "Decisões confirmadas" item 4): CAMPEONATO bloqueia
de `hora_inicio` até `hora_fim` **ou até o fechamento da unidade**, o que for maior (evita deixar a
quadra "livre" no papel enquanto o campeonato ainda pode estar rolando à noite); RANKING bloqueia
só a janela exata `hora_inicio`–`hora_fim`.

### C-5 — de onde vêm `id_campeonato` / `id_ranking`? — **RESOLVIDO (opção a)**

Não existe MS de campeonatos ativo. Opções (histórico da análise original):
- (a) **`agendamentos` cria uma entidade `campeonato` leve** (nome, datas, modalidade) que **possui
  N agendamentos** — igual a `mensalista` → `mensalista_plano` na task 004. `id_campeonato` é o
  `_id` desse documento.
- (b) `id_campeonato`/`id_ranking` são **strings opacas** fornecidas pelo front (sem entidade pai
  em `agendamentos`); "agendamento em massa" recebe a lista pronta.

A frase *"quando o usuário criar um campeonato e gerar as partidas"* sugere (a). **Confirmado:
opção (a)** — ver "Decisões confirmadas" item 6.

### C-6 — "ranking" — o que é? — **RESOLVIDO**

**Confirmado (2026-09-09):** RANKING tem dois modos —
1. **Partida marcada** de uma liga (interna da Beach Center ou de outro ranking) — é um
   agendamento com bloqueio, `lado_a`/`lado_b`, e **usa a mesma auditoria de troca de dia do
   CAMPEONATO** (só a regra de horário de bloqueio muda — item 4).
2. **"Ranking geral"** — evento por período (`data_inicio`/`data_fim`), **sem** agendamento/
   bloqueio nenhum: durante o período, o usuário reserva a quadra pelo fluxo comum de
   `scheduling`. Nesta task, "ranking geral" é **só cadastro** (entidade leve), sem lógica de
   conflito associada.

## Regras de negócio conhecidas (o que já foi informado)

1. CRUD completo (`create`/`read`/`update`/`delete`/`list`) — provável ADMIN-only (como
   `events_scheduled`).
2. **Agendamento em massa** — criar vários agendamentos de campeonato numa única chamada.
3. **Delete em massa** — remover vários (provavelmente "todos de um `id_campeonato`/`id_ranking`").
4. **Trocar o dia** — endpoint dedicado **em ambos os domínios** (campeonato **e** partida marcada
   de ranking — confirmado no item 7a das decisões):
   - recebe `dia_novo` + `motivo`;
   - grava auditoria (`usuario_nome` do token, `motivo`, `dia_anterior`, `dia_novo`, timestamp);
   - re-libera os `scheduling`s do dia antigo e bloqueia os do dia novo.
   - **Não se aplica** ao "ranking geral" (item 7b) — esse não tem agendamento para trocar de dia.
5. Criar um agendamento de campeonato **bloqueia a quadra** na(s) data(s) — `scheduling.available =
   false` (equivalente a `applyEventToSchedulings`).
6. Deletar/cancelar → **libera** os `scheduling`s afetados sem reserva ativa e não bloqueados por
   outro bloqueador (equivalente a `releaseEventFromSchedulings`).
7. Não permitir **reservar** uma quadra que tem campeonato/ranking na mesma data (via
   `EventConflictService` — a reserva/scheduling passa a enxergar as novas fontes).
8. **Regra de composição de partida — DENTRO do escopo** (C-3 revertida): `lado_a.tipo` deve ser
   igual a `lado_b.tipo` (`DUPLA`/`SOLO`/`EQUIPE`) — validado no DTO/usecase de create e
   create-bulk. Ver C-3-bis para o formato corrigido de `IPartida`.
9. Bloqueio assimétrico por tipo (C-4 resolvido): CAMPEONATO estende até o fechamento da unidade
   quando `hora_fim` está perto/depois dele; RANKING usa a janela exata.
10. **Pagamento de RANKING (só partida marcada, não campeonato):** ao anexar comprovante,
    `ranking_agendamento.status` vira `waiting_approve` (novo valor do enum compartilhado
    `IReserveStatus`); `ranking_agendamento` ganha `numero_protocolo` único
    (`generateNumericId()`, mesmo padrão de `IReserve.number`). Sem integração com
    `beach-center-bff-pagamentos` nesta task — fluxo autocontido em `agendamentos`.

## Perguntas em aberto

### Resolvidas nesta rodada (2026-09-09)

~~1. Recorrência (data vs dia-da-semana)~~ → **data específica**, um registro por partida.
~~2. Escopo de participantes/partidas~~ → **incluídos** nesta task (ver C-3, C-3-bis).
~~4. Bloqueio dia inteiro vs janela~~ → **assimétrico**: CAMPEONATO estende até o fechamento da
unidade quando `hora_fim` está perto/depois dele, RANKING usa janela exata.
~~6. Uma coleção com `tipo` vs duas coleções~~ → **duas coleções separadas**
(`campeonato_agendamento` / `ranking_agendamento`).

### Resolvidas na 2ª rodada (2026-09-09)

~~A. Limite de fechamento no bloqueio do CAMPEONATO~~ → **confirmado**: fechamento dinâmico da
unidade (`resolveOperatingHours`), não um "22h" fixo.
~~B. Formato `lado_a`/`lado_b` + `tipo` discriminante~~ → **aceito**.
~~3. Entidade-mãe (C-5)~~ → **sim**, para campeonato **e** ranking.
~~5. Natureza do RANKING~~ → **partida marcada usa a mesma auditoria do campeonato**; existe também
um modo **"ranking geral"** (cadastro leve por período, sem bloqueio) — ver decisões confirmadas
itens 6 e 7.

### Resolvidas na 3ª rodada (2026-09-09) — praticamente fechou tudo

~~7. Quadra por agendamento~~ → **1 registro = 1 quadra** (item 11 das decisões); "várias quadras"
do campeonato = vários registros no bulk, um por quadra.
~~8. Escopo do bloqueio~~ → **quadra+unidade exatos** (item 12), nunca a unidade inteira.
~~9. Agendamento em massa~~ → **gera sozinho** a partir de `id_campeonato` + parâmetros, **ignora**
os outros bloqueios ao gerar (prioridade máxima) — ver item 13 (fluxo de conflito/`DRAFT`).
~~10. Delete em massa~~ → aceita `id_campeonato`/`id_ranking` **e/ou** lista de ids.
~~11. "Trocar o dia"~~ → **um agendamento específico** (uma partida); libera o slot antigo pra
outro usuário poder usar.
~~12. Conflito ao criar~~ → **substituído pelo item 13** das decisões (fluxo de 2 passos +
`DRAFT`), exclusivo do CAMPEONATO. `RANKING`/`mensalista_plano`/`aula_bloqueio`/reserva comum
continuam rejeitando 409 simples entre si (sem prioridade especial).
~~13. Auditoria — precisa listar~~ → **sim**.
~~14. `usuario_nome`~~ → do token Firebase autenticado.
~~15. `PATCH` comum vs troca-de-dia~~ → fica **como recomendado** (só o endpoint de troca-de-dia
mexe em `data`) — o usuário concorda que o ideal seria o `PATCH` comum também poder, mas isso
depende de uma revisão futura do fluxo de reagendamento comum, fora desta task.
~~16. `EventConflictService` ganha as novas fontes~~ → **sim**, confirmado — objetivo explícito do
usuário é o kernel antigo (`RecurringBlockerSource`/dia-da-semana) ficar cada vez mais obsoleto
conforme as novas tabelas (por data) assumem.
~~19. Autorização ADMIN-only~~ → **sim, exceto o endpoint de anexar comprovante do RANKING**
(item 26 — é a pessoa que fez o agendamento, não ADMIN).
~~20. Soft-delete~~ → **`status: CANCELLED`**, sem campo `deleted` separado.
~~21. `modalidade`~~ → **string livre**.
~~22. Progressão de status do ranking~~ → confirmado **`pending` → `waiting_approve` →
`approved`/`rejected`**.
~~23. Armazenamento do comprovante~~ → **pasta num Google Drive** (mesma ideia do
`google-drive-file-storage.adapter.ts` de `beach-center-bff-pagamentos`) — **decisão de
integração exata** (reusar aquele adapter vs. credenciais/pasta próprias de `agendamentos`) fica
para confirmar numa próxima task/no `/speckit-plan`.
~~24. Endpoint de revisão (aprovar/rejeitar)~~ → **fora desta task**; fica parado em
`waiting_approve` até uma task futura "revisitar todos os endpoints existentes".
~~25. Unicidade do `numero_protocolo`~~ → **único globalmente**, contra `ranking_agendamento` **e**
`reserva.number` — não pode colidir com nenhum protocolo de reserva comum já emitido.
~~26. Quem anexa o comprovante~~ → **a pessoa que fez o agendamento** (fluxo com QR code que leva
a uma tela de anexar PDF/imagem do comprovante — não é ADMIN). Validação automática (IA) do
comprovante é **ideia para task futura**, fora de escopo agora.

### Ainda em aberto — só 2 itens não-bloqueantes (o resto fechou)

17. **`events_scheduled` `OUTRO`** — continua existindo para eventos pontuais genéricos (já
    confirmado pelo próprio usuário na task: *"falta os outros e campeonatos"* — "outros" =
    `OUTRO` fica). *(Sem ação — só reafirmando escopo.)*
18. **Migração** — confirmado que **não há** (C-1): nenhum `OUTRO` legado vira campeonato
    automaticamente.

> Ambos os itens acima já estão de fato resolvidos (herdados da 1ª rodada) — mantidos aqui só como
> registro de escopo, não bloqueiam o `/speckit-plan`.

## Impacto arquitetural previsto (Arquitetura Hexagonal, Princípio II)

Tudo em `services/beach-center-bff-agendamentos/src/` (sujeito às respostas acima):

### Novo(s) domínio(s) — `campeonato_agendamento` / `ranking_agendamento` (duas coleções — item 3
das decisões confirmadas; nomes finais a fechar no `/speckit-plan`)

Cada domínio replica o padrão de `mensalista_plano`/`aula_bloqueio` (task 004), mas com `data`
específica (não `day_of_week`) e dados de **partida** embutidos (decisão #2 — C-3 revertida):

- `domain/models/`:
  - `campeonato.model.ts` / `ranking.model.ts` — **entidades-mãe leves** (item 6 das decisões:
    nome, modalidade). `ranking.model.ts` precisa suportar os **dois modos** confirmados (item 7):
    ranking com partidas agendadas, ou "ranking geral" (`data_inicio`/`data_fim`, sem agendamentos
    — item 7b). Decisão de schema exata (campo discriminante `modo: 'PARTIDAS' | 'GERAL'`? ou dois
    models?) fica para o `/speckit-plan`.
  - `campeonato-agendamento.model.ts` / `ranking-agendamento.model.ts` — a "partida marcada"
    (item 7a): `partida: { lado_a: IParticipante; lado_b: IParticipante }` (`IParticipante` =
    `IDupla | ISolo | IEquipe` com `tipo` discriminante — C-3-bis), `data`, `hora_inicio`,
    `hora_fim`, `court`, `unit`, `modalidade`, `status`, `id_campeonato`/`id_ranking` (ref. à
    entidade-mãe).
    - **`campeonato-agendamento.model.ts`** ganha um status extra **`DRAFT`** (item 13 das
      decisões) — criado quando o admin opta por "não cancelar" os conflitos detectados; um
      `DRAFT` **não participa** do `EventConflictService`/bloqueio de `scheduling` até virar
      confirmado.
    - **Só `ranking-agendamento.model.ts`** ganha, além disso, `numero_protocolo: string` (**único
      globalmente** — item 25, contra `ranking_agendamento` e `reserva.number` juntos) e reaproveita
      o `IReserveStatus` estendido com `'waiting_approve'` (item 10) + referência ao comprovante
      anexado (PDF ou imagem — Google Drive, item 23).
  - `campeonato-agendamento-auditoria.model.ts` / `ranking-agendamento-auditoria.model.ts` — troca
    de dia, **simétrica nos dois domínios** (item 7a: ranking usa a mesma auditoria do campeonato).
- `domain/ports/input/` — input ports dos usecases (CRUD + massa + trocar-dia + auditoria), um
  conjunto por domínio (campeonato e ranking).
- `domain/ports/output/` — persistence ports (um por verbo/ação, incl. CRUD da entidade-mãe) +
  port compartilhado p/ o kernel de conflito, com matching **por data** (não dia-da-semana — item
  16). "Ranking geral" **não** entra nesse matching (sem bloqueio — item 7b). Mais um port de
  **checagem de unicidade global do protocolo** (consulta `ranking_agendamento` + `reserva` — item
  25).
- `domain/usecases/`, por domínio:
  - `create/read/update/delete/list`, `delete-bulk`, `change-day` (com auditoria), `list-auditoria`,
    mais CRUD simples da entidade-mãe (`campeonato`/`ranking`, incl. "ranking geral" sem lógica de
    bloqueio).
  - **`create-bulk` do CAMPEONATO tem um fluxo próprio, de 2 passos** (item 13): (1) recebe
    `id_campeonato` + parâmetros de geração, monta os agendamentos candidatos e **verifica
    conflitos** (`mensalista_plano`/`aula_bloqueio`/`OUTRO`/reserva comum/`ranking_agendamento`)
    sem checar prioridade nenhuma — devolve a lista de conflitos encontrados; (2) uma segunda
    chamada confirma com `cancelar_conflitos: boolean` — `true` cancela/libera tudo que conflitava
    e cria os agendamentos **confirmados**; `false` cria como **`DRAFT`** (sem bloquear nada).
    Formato exato do contrato (2 endpoints? 1 endpoint com 2 fases via id de "preview"?) é decisão
    do `/speckit-plan`.
  - `create`/`create-bulk` do RANKING (e o `create` unitário do campeonato) validam
    `lado_a.tipo === lado_b.tipo` (regra de composição), validam quadra, checam conflito **simples
    (rejeita 409, sem o fluxo de prioridade acima — isso é exclusivo do campeonato)** e aplicam o
    bloqueio **assimétrico** (item 4: campeonato estende até o fechamento da unidade, ranking usa
    janela exata — reaproveitar `SchedulingWindowValidator.resolveOperatingHours`).
  - `delete`/`delete-bulk`/`change-day` liberam schedulings (item 11: libera o slot antigo pra
    outro usuário poder usar).
  - **`ranking`**: usecase de **anexar comprovante** — recebe o arquivo (PDF/imagem), sobe pro
    Google Drive (item 23), grava a referência e transiciona `status` para `waiting_approve`.
- `infra/schemas/` — schemas Mongoose das novas coleções (com subdocumento de partida).
- `infra/adapters/` — um adapter por verbo/ação, por domínio, + adapter de matching-por-data para
  o kernel.
- `applications/dto/` — DTOs yup: validação de `data`, `partida.lado_a`/`lado_b` (união
  discriminada por `tipo`), `motivo` obrigatório na troca de dia, e DTO simples da entidade-mãe
  (incl. `data_inicio`/`data_fim` para "ranking geral").
- `applications/controllers/` — thin; o de troca-de-dia extrai `usuario_nome` do request
  autenticado. `ranking-agendamento` ganha um controller extra de **anexar comprovante**,
  **rota pública** (item 26: é a pessoa que fez o agendamento, via QR code, não ADMIN) —
  transiciona `status` para `waiting_approve`.
- `applications/routes/` — novas rotas (montadas em `routes.ts`). **ADMIN-only** em tudo, **exceto**
  a rota pública de anexar comprovante do `ranking-agendamento` (item 19/26).

### Alterações em código existente (`agendamentos`)
- `domain/usecases/shared/event-conflict.service.ts` — nova(s) fonte(s) `CAMPEONATO`/`RANKING`;
  **suporte a matching por data específica** (hoje só dia-da-semana).
- `domain/models/events-scheduled.model.ts` — `RecurringBlockerSource` +
  `'CAMPEONATO' | 'RANKING'` (e talvez renomear para `BlockerSource`).
- `domain/models/reserva.model.ts` — `IReserveStatus` ganha o valor **`'waiting_approve'`** (item
  10 das decisões confirmadas). É uma mudança em tipo **compartilhado** com a reserva comum, mas
  nesta task **nenhum usecase da reserva comum muda de comportamento** — só
  `ranking-agendamento` efetivamente usa o novo valor. Avaliar no `/speckit-plan` se cabe uma
  constante/type dedicado (`IRankingAgendamentoStatus = IReserveStatus`) em vez de importar
  `IReserveStatus` direto, para não acoplar os dois domínios além do necessário.
- `domain/ports/input/day.input-port.ts` — `IExceptionConflictEvent.source` ganha os novos valores.
- `domain/usecases/day/list-day-schedulings/list-day-schedulings.usecase.ts` — propaga `source`.
- `domain/usecases/shared/event-scheduling-impact.service.ts` — hoje itera por dia-da-semana;
  generalizar para aceitar bloqueio **por data**.
- `config/container.ts` — wiring dos novos usecases/adapters + injeção da nova fonte no
  `EventConflictService`.

## Fora de escopo (a confirmar)

- ~~MS de campeonatos (`services/beach-center-bff-campeonatos`) — não tocado; nenhuma chamada feita
  a ele.~~ **Superado pela Atualização 2** (topo do arquivo) — o MS de campeonatos passou a ser o
  dono de `campeonato`/`ranking`/`partida` e a chamar `agendamentos` via rota interna. Ver
  `plan.md`, Fase 7.
- **Geração do chaveamento** do campeonato (quem joga contra quem, fases, classificação) — isso é
  regra de campeonato/torneio, não de agendamento. Esta task só recebe a partida já definida
  (`lado_a`/`lado_b`) e agenda a quadra para ela (ver pergunta #9).
- **Integração com `beach-center-bff-pagamentos`** para o comprovante de ranking (item 10) — fica
  autocontido em `agendamentos` nesta task; se/quando precisar de checkout real, conciliação
  financeira ou aparecer no histórico de transações, é task futura.
- **Aprovação/rejeição do comprovante de ranking** — só se a pergunta #24 confirmar que entra
  nesta task; senão, fica em `waiting_approve` aguardando uma task futura.
- Front-end (`beach-center-app`).
- `events_scheduled` `OUTRO` — permanece como está (última fatia da tabela genérica).
- Cancelamento de ocorrência pontual de um agendamento recorrente (não se aplica — campeonato já é
  por data).
