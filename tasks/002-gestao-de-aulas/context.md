# Task 002 — Sistema de Gestão de Aulas

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.

## Título

Sistema de agendamento e gestão de aulas (beach tennis / vôlei / outras modalidades), com
professores gerenciando alunos e vagas, e integração com o microsserviço de agendamentos para
bloquear a quadra durante o horário da aula.

## Descrição (a partir do input do usuário)

- CRUD normal de `Aula` (create/read/update/delete/list).
- Uma aula tem: classe/modalidade, dias da semana recorrentes, horário de início/fim, professor
  responsável, quadra, e uma lista de alunos.
- Um aluno tem: nome, telefone, vencimento de fatura (mensalidade) e um flag `ativo`.
- `ativo` deve ser recalculado de forma "preguiçosa" (lazy) — validado/atualizado quando o aluno
  aparece numa listagem — no mesmo padrão que `beach-center-bff-agendamentos` usa hoje para
  marcar `scheduling.available = false` em horários passados (`markPastSchedulingsUnavailable`,
  chamado no início dos usecases de listagem).
- O agendamento da **quadra em si** continua sendo responsabilidade do
  `beach-center-bff-agendamentos` — o professor é o "usuário responsável" por esse agendamento.
- Uma aula ativa deve **impedir** que a quadra seja agendada por outra pessoa naquele
  dia-da-semana/horário. Hoje o `agendamentos` já resolve exatamente esse tipo de bloqueio através
  do recurso `events_scheduled` ("eventos agendados"), que gera conflito (`EventConflictError`)
  e bloqueia (`available=false`) os `scheduling`s sobrepostos via
  `EventSchedulingImpactService.applyEventToSchedulings`. O pedido do usuário é usar esse mesmo
  mecanismo — "estamos removendo a ideia da tabela de exceção e criando separadas para gerar
  essas exceções dentro de agendamentos" — ou seja: **ao criar uma aula, o serviço de aulas deve
  chamar o `agendamentos` para criar o(s) evento(s) correspondente(s)**, e ao cancelar/remover a
  aula, cancelar o(s) evento(s) (o que já libera os `scheduling`s via
  `releaseEventFromSchedulings`, automaticamente, sem lógica nova em `agendamentos`).
- Precisa de uma forma de cancelar a aula (ou uma ocorrência dela) de modo que a quadra volte a
  ficar disponível para agendamento nesse horário específico.

## Interfaces fornecidas pelo usuário (ponto de partida, não fechado)

```ts
interface IAlunos {
    nome: string;
    telefone: string;
    vencimento_fatura: string;
    ativo: boolean;
}

interface IAula {
    classe: string
    alunos: IAlunos[]
    dia: string[]; // dias da semana [segunda, terça, quarta, quinta, sexta, sábado, domingo]
    horaInicio: Date;
    horaFim: Date;
    professor: string;
    quadra: string;
    modalidade: string;
}
```

## Serviço(s) alvo (Princípio I)

- **`services/beach-center-bff-aulas`** — repositório já existe no ecossistema, hoje **vazio**
  (só `package.json`, sem `src/`). É o dono do domínio de aula/aluno/vaga. Precisa ser bootstrapado
  do zero seguindo a Arquitetura Hexagonal (Princípio II) e a Fase 0 já validada em
  usuarios/pagamentos/agendamentos (ESLint flat config, Jest, `config/env.ts`,
  `config/container.ts`, `main.ts`).
- **`services/beach-center-bff-agendamentos`** — **não deve ganhar lógica de negócio de aula**.
  Só é consumido via HTTP pelo `aulas` (mesmo padrão inter-serviço já usado entre
  `pagamentos` ↔ `agendamentos`: `x-api-key` + `internalApiKeyMiddleware`/`AGENDAMENTOS_INTERNAL_API_KEY`)
  para criar/cancelar `events_scheduled`. Nenhuma mudança estrutural prevista aqui, **exceto**
  se a resposta às perguntas em aberto #5/#6 abaixo exigir um ajuste no modelo de
  `events_scheduled` (hoje `day_of_week: string`, um único dia por evento).
- **`services/beach-center-bff-usuarios`** — só é afetado **se** "professor" for modelado como um
  usuário do sistema com login (ver pergunta #2). Hoje `UserType = "CLIENTE" | "ADMIN"`, sem
  `"PROFESSOR"`.

## Regras de negócio conhecidas

1. CRUD padrão de aula.
2. Aula bloqueia agendamento de quadra no(s) mesmo(s) dia(s)-da-semana/horário/quadra.
3. Cancelamento da aula deve liberar a quadra para agendamento normal.
4. Professor é o responsável pelo agendamento de quadra gerado pela aula.
5. Aluno tem `ativo` recalculado de forma lazy na listagem, a partir de `vencimento_fatura`
   (mensalidade em atraso → `ativo=false`).
6. Reaproveitar o mecanismo de `events_scheduled` já existente em `agendamentos` como a
   "exceção" que bloqueia a quadra — não criar um segundo mecanismo de exceção.

## Impacto arquitetural previsto (Arquitetura Hexagonal, Princípio II)

`services/beach-center-bff-aulas` (novo, do zero):
- `applications/controllers` + `routes` + `dto`: CRUD de aula, CRUD/gestão de aluno dentro da
  aula (adicionar/remover aluno, gerenciar vaga), listagem com o recalculo lazy de `ativo`.
- `domain/models`: `IAula`, `IAluno` (nomes finais a definir).
- `domain/usecases`: `create/read/update/delete/list-aula`, usecases de gestão de aluno, e o
  usecase de sincronização com `agendamentos` (criar/cancelar evento ao criar/cancelar aula).
- `domain/ports/output`: persistência (Mongo) + um port `IAgendamentosEventClient` (ou similar)
  para a chamada HTTP ao `agendamentos`.
- `infra/adapters`: Mongo adapters + adapter HTTP (axios) que fala com
  `agendamentos:/api/v1/eventos-agendados`.
- `config`: `env.ts` (incl. `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY` — nomes a
  confirmar/reaproveitar dos já usados em `pagamentos`), `container.ts`, `firebase.ts` (se aula
  tiver rotas autenticadas via Firebase, a confirmar).

`services/beach-center-bff-agendamentos`: nenhuma mudança prevista, a menos que a pergunta #5
exija estender `events_scheduled` para múltiplos dias por evento.

## Decisões (respondidas pelo usuário — task pronta para `/speckit-plan`)

1. **Modelo de dados do aluno — coleção separada.** `Aluno` é um documento próprio (não embutido),
   referenciando `aula_id`:
   ```ts
   interface IAula {
     id: string;
     classe: string;       // nome/identificador da turma (distinto de modalidade, decisão #8)
     modalidade: string;   // categoria fixa, ex.: "Beach Tenis" | "Volei"
     dias: string[];       // dias da semana
     hora_inicio: Date;
     hora_fim: Date;
     professor: string;    // referencia um usuario (usuarios), decisao #2
     quadra: string;       // referencia uma quadra em agendamentos (id, sem integracao ainda — decisao #3 dos deferidos)
     capacidade_maxima: number;
   }
   interface IAluno {
     id: string;
     aula_id: string;
     nome: string;
     telefone: string;
     vencimento_fatura: Date | string;
     ativo: boolean;
   }
   ```
2. **Professor é um usuário real do sistema.** Cria-se `UserType = "PROFESSOR"` em
   `beach-center-bff-usuarios` (hoje só `"CLIENTE" | "ADMIN"`). Login/autenticação via Firebase,
   mesmo padrão já usado pelos demais tipos de usuário.
3. **Vagas com limite explícito.** `capacidade_maxima: number` na aula; matricular um aluno novo
   valida `total_de_alunos_ativos_na_aula < capacidade_maxima` (a definir no `/speckit-plan` se a
   contagem considera só alunos `ativo=true` ou todos).
4. **Autorização.** `ADMIN` gerencia tudo. O próprio professor autenticado também pode gerenciar
   (matricular/remover aluno, editar vagas) das aulas onde ele é o professor responsável —
   precisa de um middleware `requireOwnerOrAdmin` (compara `req.user`/`req.databaseUser` com o
   `professor` da aula) além do `requireRole`/`authMiddleware` já padronizados nos outros
   serviços.
5. **`vencimento_fatura` é controle manual, sem integração com `pagamentos`.** ADMIN (ou o
   professor, a confirmar no plan) atualiza a data manualmente ao registrar o pagamento da
   mensalidade da aula. A regra de recálculo lazy (`hoje > vencimento_fatura → ativo=false`,
   reavaliada na listagem de alunos, no mesmo padrão de `markPastSchedulingsUnavailable` de
   `agendamentos`) permanece confirmada.

## Fora de escopo nesta task (adiado explicitamente pelo usuário)

- **Integração com `agendamentos`/`events_scheduled` para bloquear a quadra durante a aula.** O
  usuário está reestruturando `events_scheduled` (hoje um modelo genérico com
  `event_type: 'AULA_BEACH_TENIS' | 'MENSALISTA' | 'AULA_VOLEI' | 'OUTRO'`) em tabelas/modelos
  separados por domínio (aulas, campeonatos, mensalista) dentro de `agendamentos`. Essa
  refatoração ainda não está pronta. **Portanto, nesta task 002, `aulas` NÃO chama
  `agendamentos`, não cria/cancela eventos, e não gera um `events_scheduled` por dia da semana.**
  O campo `quadra` na aula fica só como referência/informação (id da quadra), sem nenhuma
  validação de conflito nem bloqueio automático de agendamento. Isso será resolvido numa task
  futura, depois que a separação de `events_scheduled` em `agendamentos` estiver concluída.
  Consequentemente, também ficam fora do escopo: geração de exceção pontual (cancelamento de uma
  ocorrência específica da aula) e qualquer chamada HTTP `aulas → agendamentos`.
- **Qualquer integração com `beach-center-bff-pagamentos`** para a mensalidade de aula (decisão
  #5 acima).

## Impacto arquitetural (atualizado — sem integração externa nesta rodada)

`services/beach-center-bff-aulas` (novo, do zero, bootstrapar a Fase 0 já validada em
usuarios/pagamentos/agendamentos: ESLint flat config, Jest, `config/env.ts`,
`config/container.ts`, `main.ts`):
- `applications/controllers` + `routes` + `dto`: CRUD de aula (ADMIN), CRUD de aluno dentro de
  uma aula (ADMIN ou professor dono da aula), listagem de alunos com recálculo lazy de `ativo`.
- `domain/models`: `IAula`, `IAluno`.
- `domain/usecases`: create/read/update/delete/list de aula; create/read/update/delete/list de
  aluno (com validação de `capacidade_maxima` na matrícula); usecase de listagem de alunos com o
  recálculo lazy de `ativo`.
- `domain/ports/output`: persistência Mongo de `aula` e `aluno`. Sem port HTTP para outro
  microsserviço nesta rodada (o professor é validado localmente pelo token/role já resolvido
  pelo `authMiddleware`, sem precisar chamar `usuarios` de volta — a confirmar no plan se precisa
  buscar dados adicionais do professor).
- `infra/adapters`: Mongo adapters apenas.
- `config`: `env.ts`, `container.ts`, `firebase.ts` (rotas autenticadas via Firebase, mesmo
  padrão dos outros serviços).

`services/beach-center-bff-usuarios`: adicionar `"PROFESSOR"` a `UserType` (`domain/models/user.model.ts`,
`infra/schemas/user.schema.ts`) e qualquer usecase/dto que hoje valide o enum de `user_type`
explicitamente.

`services/beach-center-bff-agendamentos`: **nenhuma mudança nesta task.**

## Observação sobre o estado atual do repositório

A task 001 (`tasks/001-backend-conformidade-hexagonal`) está sendo finalizada em paralelo — a
fase 3 (`beach-center-bff-agendamentos`, incluindo a integração final do `config/container.ts`)
está em andamento. Como a task 002 não toca `agendamentos`, ela pode ser planejada e implementada
independentemente, sem esperar a 001 terminar.
