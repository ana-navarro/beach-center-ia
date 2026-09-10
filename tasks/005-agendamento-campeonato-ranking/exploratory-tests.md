# Testes Exploratórios Manuais — Task 005 (Agendamento de Campeonato e Ranking)

> Gerado por `/speckit-test`. Roteiro de QA manual (o projeto não tem analistas de QA dedicados).
> **Não** contém código de teste automatizado — esses estão em `/speckit-unit-tests`
> (agendamentos 275 suites / 1289 testes; campeonatos 83 / 329).
> Base: `plan.md` (Critérios de Aceite AC-1..AC-28, já ajustados pela "Revisão arquitetural").

---

## 1. Escopo e pré-condições

### 1.1 Serviços envolvidos

| Serviço | Papel no teste | Auth | Base URL |
|---|---|---|---|
| `beach-center-bff-campeonatos` | **Entrada do ADMIN.** Dono de `campeonato`/`ranking`/`partida`. Relaia o bloqueio de quadra pra `agendamentos`. | Firebase ID token + `role=ADMIN` | `http://localhost:<PORT_CAMPEONATOS>/api/v1` |
| `beach-center-bff-agendamentos` | Dono do bloqueio de quadra (`campeonato_agendamento`/`ranking_agendamento` + auditorias + kernel de conflito). | `x-api-key: <AGENDAMENTOS_INTERNAL_API_KEY>` nas rotas internas; **1 rota pública** (comprovante) sem auth | `http://localhost:<PORT_AGENDAMENTOS>/api/v1` |
| `beach-center-bff-usuarios` | Emissão/validação de token ADMIN. | — | — |
| `beach-center-bff-pagamentos` | **Só regressão** — a task NÃO integra com ele (comprovante é autocontido em `agendamentos`). | — | — |

> ⚠️ **Mudança arquitetural vs. texto original de alguns AC:** não existe fluxo
> `verificar`/`confirmar` em 2 endpoints. O `POST /api/v1/campeonato-agendamentos` (interno) usa
> o padrão **"tenta → 409 com `conflitos[]` → resubmete a MESMA chamada com `cancelar_conflitos`"**
> (espelha `CloseDateUsecase`). As entidades-mãe (`/campeonatos`, `/rankings`, `/partidas`) vivem
> **no MS de campeonatos**, não em `agendamentos`. As rotas de agendamento de campeonato/ranking
> são **internas (`x-api-key`)**, não ADMIN/Firebase.

### 1.2 Ambiente

- Subir o ecossistema com `docker-compose.dev.yml` (task 003) ou `npm run dev` em cada serviço.
- Mongo limpo (ou base de staging isolada) — vários cenários criam/cancelam bloqueios de quadra.
- Variáveis obrigatórias:
  - `agendamentos`: `AGENDAMENTOS_INTERNAL_API_KEY` (senão TODA rota interna responde `401`).
  - `campeonatos`: `AGENDAMENTOS_API_URL` + `AGENDAMENTOS_INTERNAL_API_KEY` (senão `agendar`/`trocar-dia` retornam `500 "Integração com agendamentos não configurada"`).
  - `campeonatos`: `DB`, `FIREBASE_PROJECT_ID`.
- Variáveis **opcionais** (comprovante): `GOOGLE_DRIVE_CLIENT_EMAIL`, `GOOGLE_DRIVE_PRIVATE_KEY`, `GOOGLE_DRIVE_FOLDER_ID`. Testar **os dois modos**: configuradas (upload real no Drive) e ausentes (fallback base64 no documento).

### 1.3 Dados de seed

| Item | Detalhe |
|---|---|
| Usuário ADMIN | token Firebase válido, `role=ADMIN` no `beach-center-bff-usuarios`. |
| Usuário comum | token válido, **sem** `role=ADMIN` (para os testes de autorização). |
| Unidade A | `unit` cujo fechamento é **22:00** — usar o id `6a440a931094fad2f585011b` (hard-coded em `SchedulingWindowValidator.resolveOperatingHours`). |
| Unidade B | qualquer outra `unit` (fechamento **23:00**). |
| Quadras | ≥ 2 quadras na Unidade A (`court` 1 e 2) + ≥ 1 quadra na Unidade B. |
| `mensalista_plano` ativo | recorrente, p.ex. "todo sábado 19:00–20:00", Unidade A / quadra 1. |
| `aula_bloqueio` ativo | recorrente, p.ex. "toda quinta 18:00–19:00", Unidade A / quadra 1. |
| `events_scheduled` `OUTRO` `CONFIRMED` | pontual/recorrente numa data de teste. |
| Reserva comum ativa | um `scheduling` com reserva `approved`/`pending` na Unidade A / quadra 2, numa data de teste. |
| Datas de teste | escolher um **sábado futuro** `D1` e o **sábado seguinte** `D2` (para AC-2). Nunca usar data passada. |

### 1.4 Convenções deste roteiro

- IDs: `HP-*` caminho feliz · `EX-*` exceção · `ED-*` edge case · `RG-*` regressão.
- "via ADMIN" = request ao MS de campeonatos com token Firebase. "via interna" = request direto ao MS de agendamentos com `x-api-key` (para isolar a camada).
- Coluna **AC** = Critério(s) de Aceite coberto(s). ✅auto = há teste automatizado equivalente; ⚠️manual = só verificável aqui.

---

## 2. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado | AC |
|---|---|---|---|---|
| **HP-01** | ADMIN autenticado; nenhum campeonato criado | `POST /api/v1/campeonatos` `{nome, modalidade}` → `GET /:id` → `PATCH /:id` (muda `nome`) → `GET` (lista) | `201` com `status:"ATIVO"`; `GET` devolve o registro; `PATCH` reflete a mudança; lista contém o campeonato | AC-4 / AC-24 ✅auto |
| **HP-02** | ADMIN autenticado | `POST /api/v1/rankings` `{nome, modalidade, modo:"PARTIDAS"}` e outro `{nome, modalidade, modo:"GERAL", data_inicio:"2026-11-01", data_fim:"2026-11-30"}` | Ambos `201`, `status:"ATIVO"`. O `GERAL` guarda as datas; o `PARTIDAS` não | AC-4 / AC-21 / AC-24 ✅auto |
| **HP-03** | HP-02 (ranking `GERAL` criado) | Verificar `ranking_agendamento`s gerados; tentar reservar quadra no período via fluxo comum de `scheduling`/reserva | **Nenhum** `ranking_agendamento` criado; **nenhum** `scheduling` marcado indisponível; reserva comum no período funciona normalmente | AC-21 ✅auto |
| **HP-04** | Campeonato `ATIVO` (HP-01); 3 partidas `DUPLA×DUPLA` `ATIVA` sem `id_agendamento`; Unidade A / quadra 1; data `D1` livre | `POST /api/v1/campeonatos/:id/agendar` `{quadras:[court1], datas:[D1], hora_inicio:"14:00", duracao_partida_minutos:60}` (via ADMIN) | `201`; `agendamentos` recebeu 1 `create-bulk` com `quantidade:3`; 3 `campeonato_agendamento` `CONFIRMED` criados (14:00–15:00, 15:00–16:00, 16:00–17:00); cada `partida` ligada ao `id_agendamento` correspondente por índice; os 3 `scheduling`s viram `available:false` | AC-25 / AC-5 / AC-7(sem conflito) ✅auto |
| **HP-05** | HP-04 concluído | Repetir **a mesma** chamada `POST /campeonatos/:id/agendar` sem criar novas partidas | `400`/no-op — "Nenhuma partida pendente de agendamento"; nenhum `campeonato_agendamento` novo (idempotência) | AC-25 ✅auto |
| **HP-06** | Ranking `modo:"PARTIDAS"` `ATIVO` (HP-02); 2 partidas `SOLO×SOLO` `ATIVA` sem `id_agendamento`; quadra livre | `POST /api/v1/rankings/:id/agendar` `{quadras, datas:[D1], hora_inicio:"20:00", duracao_partida_minutos:60}` | `201`; 2 `ranking_agendamento` `status:"pending"` criados com **`numero_protocolo`** único cada; bloqueio aplicado na **janela exata** (20:00–21:00, 21:00–22:00); partidas ligadas | AC-16(sem conflito) / AC-17 / AC-18 ✅auto |
| **HP-07** | 1 `campeonato_agendamento` `CONFIRMED` (D1) da HP-04; data `D2` livre na mesma quadra/horário | `PATCH /api/v1/partidas/:id/trocar-dia` `{dia_novo:D2, motivo:"chuva prevista"}` (via ADMIN) | `200`; slot de `D1` liberado (`scheduling.available=true` onde elegível), slot de `D2` bloqueado; registro passa a `data=D2`; **1 auditoria** criada em `campeonato_agendamento_auditorias` com `usuario_nome` (= `req.databaseUser.name` do ADMIN), `motivo`, `dia_anterior=D1`, `dia_novo=D2`, timestamp | AC-12 / AC-28 ✅auto |
| **HP-08** | HP-07 (1 troca de dia feita) + fazer uma 2ª troca | `GET /api/v1/campeonato-agendamentos/auditoria?agendamento_id=<id>` (via interna, `x-api-key`) | `200` com **2** entradas, cada uma com `usuario_nome`/`motivo`/`dia_anterior`/`dia_novo` | AC-15 ✅auto |
| **HP-09** | `ranking_agendamento` `status:"pending"` da HP-06, com `numero_protocolo` conhecido; sem token | `PATCH /api/v1/ranking-agendamentos/protocolo/:numero_protocolo/comprovante` `{file_name:"comp.pdf", mime_type:"application/pdf", base64:"<...>"}` | `200`; `comprovante` salvo (Google Drive se env configurada — `storage:"google_drive"`, `view_url`/`preview_url`; senão `storage:"database"`, `data` base64); `status` vira `waiting_approve` | AC-19 ✅auto |
| **HP-10** | ranking `modo:"PARTIDAS"` com partidas agendadas; ranking `ATIVO` | `PATCH /api/v1/rankings/:id/delete` (via ADMIN) | `agendamentos` recebe `delete-bulk` **sem `ids`** → todos os `ranking_agendamento` daquele `id_ranking` viram `cancelled`; `scheduling`s elegíveis voltam a `available:true`; ranking `status:"CANCELADO"` | AC-27 ✅auto |
| **HP-11** | 1 campeonato com ≥ 3 partidas agendadas; cancelar **1** partida específica | `PATCH /api/v1/partidas/:id/delete` (via ADMIN) | `agendamentos` recebe `delete-bulk` com `ids:[id_agendamento]` → **só** aquele `campeonato_agendamento` vira `CANCELLED`; as outras 2 partidas seguem `CONFIRMED` e bloqueando | AC-27 / AC-14 ✅auto |
| **HP-12** | 3 `campeonato_agendamento` `CONFIRMED` do mesmo `id_campeonato` (via interna) | `PATCH /api/v1/campeonato-agendamentos/:id/delete` (1) → `PATCH /api/v1/campeonato-agendamentos/delete-bulk` `{id_campeonato}` sem `ids` (resto) | Todos `CANCELLED`; todos os `scheduling`s elegíveis voltam a `available:true` | AC-14 ✅auto |

---

## 3. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado | AC |
|---|---|---|---|---|
| **EX-01** | `mensalista_plano` recorrente ocupando o 2º dos 3 slots que a alocação vai gerar; campeonato `ATIVO` com 3 partidas pendentes | `POST /api/v1/campeonatos/:id/agendar` **sem** `cancelar_conflitos` | `409`, corpo com `code:"CAMPEONATO_AGENDAMENTO_CONFLICT"` e `conflitos:[{candidato_index:1, tipo:"BLOQUEADOR", source:"MENSALISTA", id, data, hora_inicio, hora_fim, court}]`; **nada persistido** em `campeonato_agendamento`; `mensalista_plano` intacto | AC-6(1º passo) / AC-26 ✅auto |
| **EX-02** | EX-01 devolveu 409 | Resubmeter **a mesma** chamada + `cancelar_conflitos:false` | Os 3 candidatos são criados como `campeonato_agendamento` **`DRAFT`**; `mensalista_plano` **continua intacto e bloqueando**; **nenhum** `scheduling` muda de `available`; `DRAFT` não aparece em conflito/`exception_conflicts` | AC-6 / AC-3 ✅auto |
| **EX-03** | EX-01 devolveu 409 (partir de estado limpo) | Resubmeter + `cancelar_conflitos:true` | `mensalista_plano` conflitante vira `status:"CANCELLED"` (quadra liberada onde aplicável); os 3 candidatos nascem `CONFIRMED`; os `scheduling`s correspondentes viram `available:false` | AC-7 / AC-26 ✅auto |
| **EX-04** | slot em conflito com um **`ranking_agendamento`** (fonte `RANKING`) — criar 1 ranking_agendamento CONFIRMED e depois agendar campeonato no mesmo slot | `POST /campeonatos/:id/agendar` + `cancelar_conflitos:true` | O `ranking_agendamento` conflitante é **cancelado** (`status:"cancelled"`, scheduling liberado) e o campeonato confirma. **[Regressão do bug corrigido no `/speckit-validate`]** — antes o `ConflictResolutionService` não tinha o usecase de cancelar ranking injetado e lançava `"Cancelamento de conflito RANKING nao configurado"` | AC-7 (fonte RANKING) ✅auto |
| **EX-05** | 5 partidas pendentes; só 1 quadra + 1 data com espaço para **2** slots antes do fechamento da unidade | `POST /campeonatos/:id/agendar` | `400` — "Quadras/datas insuficientes: 5 partida(s) para 2 slot(s) disponivel(eis)"; **não** aloca parcialmente; nada persistido | AC-8 ✅auto |
| **EX-06** | partida com `lado_a.tipo="DUPLA"` e `lado_b.tipo="EQUIPE"` | `POST /api/v1/partidas` (criar partida) no MS de campeonatos | `400` antes de qualquer persistência — validado no DTO (`yup.lazy` discriminado) **e** no usecase (`assertPartidaComposicaoValida`). Repetir com `SOLO×EQUIPE`, `DUPLA×SOLO` | AC-11 / AC-24 ✅auto |
| **EX-07** | ranking `modo:"PARTIDAS"` com 1 partida; slot já ocupado por **qualquer** bloqueador `CONFIRMED` (mensalista/aula/OUTRO/reserva ativa/campeonato) | `POST /api/v1/rankings/:id/agendar` | `409` — RANKING **não** tem fluxo de prioridade nem `DRAFT`; rejeita o **lote inteiro** (`RankingAgendamentoConflictError`), nada persistido, nenhum protocolo gasto | AC-16 ✅auto |
| **EX-08** | `ranking_agendamento` já em `waiting_approve` (após HP-09) | `PATCH /ranking-agendamentos/protocolo/:numero/comprovante` de novo | `409` "Comprovante já foi enviado para este agendamento"; comprovante anterior **não** é substituído; `status` continua `waiting_approve`. Repetir com `status` `approved`/`rejected` manualmente setado → também `409` | AC-20 ✅auto |
| **EX-09** | comprovante com `mime_type` fora da lista | `PATCH .../comprovante` `{mime_type:"application/zip", ...}` | `400` — só `application/pdf`, `image/png`, `image/jpeg` aceitos (`yup oneOf`) | AC-19 (borda) ✅auto |
| **EX-10** | protocolo inexistente | `PATCH /ranking-agendamentos/protocolo/NAO-EXISTE/comprovante` | `404` "Agendamento de ranking não encontrado para este protocolo" | AC-19 (borda) ✅auto |
| **EX-11** | troca de dia sem `motivo` | `PATCH /api/v1/partidas/:id/trocar-dia` `{dia_novo:D2}` (sem `motivo`) | `400` "motivo é obrigatório" (yup `required` no DTO + guard no usecase); nenhuma auditoria criada; registro não muda | AC-12 (borda) / AC-28 ✅auto |
| **EX-12** | `dia_novo` já tem um `aula_bloqueio` recorrente no mesmo horário; campeonato_agendamento `CONFIRMED` | `PATCH /api/v1/partidas/:id/trocar-dia` `{dia_novo, motivo}` **sem** `cancelar_conflitos` | `409` com `conflitos:[{source:"AULA", ...}]`; **nada é movido**, registro continua no dia antigo, sem auditoria. Depois resubmeter com `cancelar_conflitos:true` → cancela a aula e move (vira HP-07) | AC-13 ✅auto |
| **EX-13** | troca de dia de um `campeonato_agendamento` em `status:"DRAFT"` (ou `CANCELLED`) | `PATCH .../trocar-dia` | `400` "Só é possível trocar o dia de um agendamento CONFIRMED" | AC-13 (borda) ✅auto |
| **EX-14** | troca de dia para **data no passado** | `PATCH .../partidas/:id/trocar-dia` `{dia_novo:"2020-01-01", motivo}` | `400` "Não é possível trocar para uma data passada" | AC-12 (borda) ✅auto |
| **EX-15** | `ranking` `modo:"GERAL"` **sem** `data_inicio`/`data_fim`, ou com `data_inicio >= data_fim` | `POST /api/v1/rankings` | `400` — `GERAL` exige as duas datas e `data_inicio < data_fim` (`assertRankingModoConsistency`) | AC-4 ✅auto |
| **EX-16** | `ranking` `modo:"PARTIDAS"` **com** `data_inicio`/`data_fim` no corpo | `POST /api/v1/rankings` | `400` — `PARTIDAS` rejeita as datas de período | AC-4 ✅auto |
| **EX-17** | integração `agendamentos` fora do ar / URL errada | `POST /api/v1/campeonatos/:id/agendar` com `AGENDAMENTOS_API_URL` apontando para porta morta | Erro propagado de forma controlada (5xx), **sem** deixar `partida` com `id_agendamento` pela metade (nenhuma partida deve ser ligada se o `create-bulk` falhou) | (robustez de integração) ⚠️manual |
| **EX-18** | `AGENDAMENTOS_INTERNAL_API_KEY` divergente entre os 2 serviços | `POST /campeonatos/:id/agendar` | MS de campeonatos recebe `401` de `agendamentos` e propaga erro claro; nada é agendado/ligado | AC-22 (borda) ⚠️manual |
| **EX-19** | `partida` sem `id_agendamento` (nunca agendada) | `PATCH /api/v1/partidas/:id/trocar-dia` `{dia_novo, motivo}` | `400` "Partida ainda não foi agendada" — não chama `agendamentos` | AC-28 ✅auto |
| **EX-20** | `partida` com `status != "ATIVA"` (cancelada) | `PATCH .../partidas/:id/trocar-dia` | `400` "Só é possível trocar o dia de uma partida ATIVA" | AC-28 (borda) ✅auto |

---

## 4. Autorização

| ID | Pré-condição | Passos | Resultado esperado | AC |
|---|---|---|---|---|
| **EX-21** | sem token / token inválido | `POST|GET|PATCH` em `/api/v1/campeonatos`, `/api/v1/rankings`, `/api/v1/partidas` (MS campeonatos) | `401` em todos | AC-22 ✅auto |
| **EX-22** | token válido de **usuário comum** (não ADMIN) | mesmos endpoints | `403` em todos | AC-22 ✅auto |
| **EX-23** | sem `x-api-key` (ou chave errada) | `POST|GET|PATCH` em `/api/v1/campeonato-agendamentos` e `/api/v1/ranking-agendamentos` (MS agendamentos) | `401 "Unauthorized"` em **todas** as rotas internas (`internalApiKeyMiddleware`) | AC-22 ✅auto |
| **EX-24** | sem auth nenhuma | `PATCH /api/v1/ranking-agendamentos/protocolo/:numero/comprovante` | **Funciona** (`200`/`404`/`409` conforme o caso) — é a **única** rota pública; não exige `x-api-key` nem Firebase | AC-22 ✅auto |
| **EX-25** | token ADMIN válido mas expirado/rotacionado | qualquer endpoint ADMIN | `401` (não `500`) | AC-22 (borda) ⚠️manual |

---

## 5. Edge cases

| ID | Pré-condição | Passos | Resultado esperado | AC |
|---|---|---|---|---|
| **ED-01** | `campeonato_agendamento` `CONFIRMED` em `data:"D1"` (sábado), quadra 1 / Unidade A, 14:00–15:00 | Verificar disponibilidade / criar reserva comum em **`D2`** (sábado seguinte), mesma quadra/horário | **Sem conflito** — o bloqueio pontual só vale para `D1`. Verificar também na mesma quadra em `D1` → **conflita** | AC-2 ✅auto |
| **ED-02** | Unidade A fecha **22:00**; agendar campeonato `hora_inicio:"20:00"`, `duracao_partida_minutos:90` (partida cruzaria 21:30, e a próxima 23:00 > 22:00) | `POST /campeonatos/:id/agendar` com 1 partida | 1 slot: `20:00`–**`22:00`** (a **última** partida do dia/quadra é estendida até o fechamento). Conferir no `scheduling`: `20:00`–`22:00` indisponível | AC-10 ✅auto |
| **ED-03** | mesmo cenário de ED-02 mas 2 partidas de 60min a partir de 20:00 | agendar 2 partidas | Slot 1: `20:00`–`21:00` (fim exato — não é a última). Slot 2: `21:00`–**`22:00`** (última → estende até fechamento, coincide) | AC-10 ✅auto |
| **ED-04** | `ranking_agendamento` `hora_inicio:"20:00"`, `hora_fim:"21:00"`, Unidade A (fecha 22:00) | aplicar bloqueio (HP-06) | **Só** `20:00`–`21:00` indisponível; `21:00`–`22:00` **continua livre** (ranking usa janela exata, NÃO estende — ao contrário do campeonato) | AC-17 ✅auto |
| **ED-05** | `campeonato_agendamento` `CONFIRMED` na **quadra 1 / Unidade A** | consultar disponibilidade da **quadra 2 / Unidade A** e da **quadra 1 / Unidade B**, mesma data/horário | Ambas **livres** — matching é sempre por `court` + `unit` exatos, nunca unidade inteira | AC-9 ✅auto |
| **ED-06** | 5 fontes simultâneas de conflito no mesmo slot: `events_scheduled` `OUTRO`, `mensalista_plano`, `aula_bloqueio` (recorrentes) + `campeonato_agendamento` + `ranking_agendamento` `CONFIRMED` (pontuais), mesma quadra+unidade+data+horário | criar uma reserva comum nesse slot (dispara `EventConflictService.getConflicts`) | As **5** fontes aparecem no resultado, cada uma com `source` correto; a reserva é bloqueada. Confirmar que as 3 recorrentes se comportam **exatamente** como antes da task (rodar as suítes existentes de `event-conflict.service` / `event-scheduling-impact.service` / `list-day-schedulings`) | AC-1 / AC-23 ✅auto |
| **ED-07** | `numero_protocolo` já existe em `reservas.number` (forçar via seed) | criar `ranking_agendamento` até "sortear" (ou mockar o gerador) o número já usado | Loop regenera antes de persistir — **nunca** colide entre `reservas` e `ranking_agendamentos`; índice único no schema não estoura | AC-18 ✅auto |
| **ED-08** | agendar campeonato com **`datas`** contendo 2 datas e **`quadras`** com 2 quadras, `quantidade:4`, duração 60, início 14:00, unidade fecha 23:00 | `POST /campeonatos/:id/agendar` | Alocação determinística: itera `data` (ordem recebida) × `quadra` (ordem recebida) gerando slots; 4 candidatos preenchidos na ordem; repetir a chamada gera **exatamente** os mesmos slots (sem estado de servidor entre chamadas) | AC-5 (determinismo) / AC-25 ✅auto |
| **ED-09** | `quadras` com quadras de **unidades diferentes** | `POST /campeonatos/:id/agendar` `{quadras:[courtUnidA, courtUnidB], ...}` | `400` "Todas as quadras devem pertencer a mesma unidade" | AC-8 (borda) ✅auto |
| **ED-10** | `quantidade:0` ou negativa / `duracao_partida_minutos:0` | `POST /campeonato-agendamentos` (via interna) | `400` no DTO (`min(1)`) | AC-8 (borda) ✅auto |
| **ED-11** | fuso: datas em `"YYYY-MM-DD"`, servidor em UTC, unidade em America/Sao_Paulo (-03:00) | agendar num sábado e conferir o `day_of_week` derivado e o matching com `scheduling`s daquele dia | O `getLocalDateKey`/`getLocalDayOfWeek` usam offset `-03:00` fixo (BR sem horário de verão); o bloqueio cai no dia local correto, não "escorrega" para sexta/domingo | AC-2 / AC-10 (borda) ⚠️manual |
| **ED-12** | `update` de `campeonato_agendamento` só permite `modalidade` | `PATCH /api/v1/campeonato-agendamentos/:id` `{data:"2027-01-01"}` (via interna) | `data` é **ignorado** (stripUnknown) — trocar dia só via `/trocar-dia` | AC-12 (borda) ✅auto |
| **ED-13** | `delete-bulk` com `ids:[]` (array vazio) e com `id_campeonato` que não tem nenhum agendamento | `PATCH /campeonato-agendamentos/delete-bulk` | Sem erro; 0 registros afetados; resposta coerente (lista vazia / count 0) | AC-14 (borda) ⚠️manual |
| **ED-14** | comprovante base64 grande (~5–10 MB), sem env do Drive | `PATCH .../comprovante` | Fallback: `storage:"database"`, `data` guardado no documento. Avaliar limite de tamanho de documento Mongo (16 MB) — documentar se há truncamento/erro | AC-19 (borda) ⚠️manual |
| **ED-15** | agendar campeonato quando `hora_inicio` + duração já ultrapassa o fechamento no 1º slot | `POST /campeonatos/:id/agendar` `{hora_inicio:"21:30", duracao_partida_minutos:60}` Unidade A (fecha 22:00) | Pool de slots **vazio** → `400` "Quadras/datas insuficientes" (não cria slot que estoura o fechamento) | AC-8 / AC-10 (borda) ✅auto |

---

## 6. Checklist de regressão (fluxos vizinhos)

| ID | Área | O que verificar | Resultado esperado |
|---|---|---|---|
| **RG-01** | Reserva comum | Criar/cancelar reserva numa quadra **sem** nenhum bloqueio de campeonato/ranking | Comportamento idêntico ao de antes da task — `EventConflictService` com as 2 novas fontes vazias não altera nada |
| **RG-02** | `mensalista_plano` (task 004) | CRUD + conflito de plano de mensalista, recorrente | Sem regressão — matching por `day_of_week` intacto; `specific_date` ausente nessas fontes |
| **RG-03** | `aula_bloqueio` (task 004) | CRUD + conflito + `applyEventToSchedulings`/`releaseEventFromSchedulings` | Sem regressão — o branch `matchesEventDay` cai no caminho recorrente quando `specific_date` é ausente |
| **RG-04** | `events_scheduled` `OUTRO` | Criar/cancelar `OUTRO`; conferir que continua sendo o único valor "genérico" e bloqueia normalmente | Sem regressão — `RecurringBlockerSource` ganhou valores mas o default `"EVENT"` é preservado |
| **RG-05** | `GET /dias` / `list-day-schedulings` | Listar exceções de um dia que tem campeonato + ranking + mensalista + aula | As novas fontes aparecem com `source:"CAMPEONATO"`/`"RANKING"` em `IExceptionConflictEvent`; as antigas inalteradas |
| **RG-06** | `day/close-date` (`CloseDateUsecase`) | Fechar uma data que tem reservas | Sem regressão — o `ConflictResolutionService` novo reusa `IDeleteReserveUseCase` com `skipCancellationTimeValidation:true`, mesmo flag do `CloseDateUsecase` |
| **RG-07** | `reserva.model` / `IReserveStatus` | Fluxo de reserva comum completo (pending → approved / rejected / cancelled) | O tipo `RankingAgendamentoStatus` é **independente** (não alargou `IReserveStatus`); nenhum `switch`/`oneOf` de reserva comum rejeita ou aceita valor novo por engano |
| **RG-08** | `beach-center-bff-pagamentos` | Submeter/revisar comprovante manual de uma **reserva comum** | Sem regressão — a task **não** tocou em `pagamentos`; o comprovante do ranking é 100% autocontido em `agendamentos` (cópia do adapter, não import) |
| **RG-09** | `beach-center-bff-campeonatos` bootstrap | `GET /health` / subida do serviço / auth middleware | Serviço sobe; camada de auth (idêntica a `beach-center-bff-aulas`) valida token e `role` |
| **RG-10** | Container / DI | `npm test` de `container.spec.ts` e `routes.spec.ts` nos 2 repos | Todas as chaves novas presentes; contagem de rotas atualizada; **`ConflictResolutionService` recebe `deleteRankingAgendamentoUsecase`** (corrigido no `/speckit-validate`) |
| **RG-11** | Kernel — suítes existentes | `npx jest event-conflict event-scheduling-impact list-day-schedulings` em `agendamentos` | 100% verdes **sem** modificação (prova de retrocompatibilidade — AC-1/AC-23) |

---

## 7. Rastreabilidade AC → cenário

| AC | Descrição resumida | Cenários | Cobertura automatizada |
|---|---|---|---|
| **AC-1** | `EventConflictService` mescla 5 fontes sem regressão | ED-06, RG-11 | ✅ `event-conflict.service.spec` |
| **AC-2** | Bloqueio pontual não vaza pra outras semanas | ED-01, ED-11 | ✅ |
| **AC-3** | `DRAFT` não bloqueia nada | EX-02 | ✅ |
| **AC-4** | CRUD de `campeonato`/`ranking` (+ regras de `modo`) | HP-01, HP-02, EX-15, EX-16 | ✅ (no MS campeonatos — AC-24) |
| **AC-5** | `verificar` não persiste nada *(→ 1º passo do 409+resubmit)* | HP-04, EX-01, ED-08 | ✅ `create-bulk` spec |
| **AC-6** | `cancelar_conflitos=false` gera `DRAFT` e não mexe em nada | EX-01, EX-02 | ✅ |
| **AC-7** | `cancelar_conflitos=true` cancela o conflito e confirma | EX-03, EX-04 | ✅ |
| **AC-8** | Falta de quadra/data suficiente é rejeitada | EX-05, ED-09, ED-10, ED-15 | ✅ `slot-allocator` spec |
| **AC-9** | Bloqueio exato por quadra+unidade | ED-05 | ✅ |
| **AC-10** | Campeonato estende até o fechamento quando `hora_fim` tardia | ED-02, ED-03, ED-15 | ✅ `slot-allocator` spec |
| **AC-11** | Composição de partida — mismatch rejeitado | EX-06 | ✅ `partida-composicao.validator` spec (campeonatos) |
| **AC-12** | Trocar o dia sem conflito | HP-07, EX-11, EX-14, ED-12 | ✅ `change-day` spec |
| **AC-13** | Trocar o dia com conflito exige confirmação *(→ 409, não "200 com lista")* | EX-12, EX-13 | ✅ |
| **AC-14** | Delete individual e em massa | HP-11, HP-12, ED-13 | ✅ `delete` / `delete-bulk` spec |
| **AC-15** | Listar auditoria | HP-08 | ✅ `list-auditoria` spec |
| **AC-16** | Criar `ranking_agendamento` com conflito → 409, sem fluxo especial | HP-06, EX-07 | ✅ `create-bulk-ranking` spec |
| **AC-17** | Bloqueio do ranking é a janela exata | HP-06, ED-04 | ✅ `slot-allocator` (`extendLastSlotToClosing:false`) |
| **AC-18** | `numero_protocolo` único globalmente | HP-06, ED-07 | ✅ `is-protocol-number-taken` spec |
| **AC-19** | Anexar comprovante muda o status e é público | HP-09, EX-09, EX-10, ED-14 | ✅ `attach-proof` spec |
| **AC-20** | Reenviar comprovante depois de enviado → 409 | EX-08 | ✅ |
| **AC-21** | "Ranking geral" não gera agendamento nem bloqueio | HP-02, HP-03 | ✅ `ranking-modo.validator` spec |
| **AC-22** | Tudo autenticado, exceto o comprovante | EX-18, EX-21, EX-22, EX-23, EX-24, EX-25 | ✅ `routes.spec` / middleware specs |
| **AC-23** | Isolamento de camadas + specs do kernel (≥80% cobertura) | RG-11, ED-06 | ✅ `npm run test:coverage` (agendamentos 98.85% / campeonatos 98.63%) + lint (`no import domain→infra`) |
| **AC-24** | CRUD de `campeonato`/`ranking`/`partida` no MS correto (XOR, dono ATIVO, composição) | HP-01, HP-02, EX-06 | ✅ (MS campeonatos) |
| **AC-25** | Agendar campeonato empurra o bloqueio e liga as partidas por índice (idempotente) | HP-04, HP-05, ED-08 | ✅ `agendar-campeonato` spec |
| **AC-26** | Conflito ao agendar propaga a decisão ponta a ponta (409 + `conflitos`) | EX-01, EX-03 | ✅ `agendar-campeonato` + client adapter spec |
| **AC-27** | Cancelar libera o bloqueio no serviço certo (`delete-bulk` com/sem `ids`) | HP-10, HP-11 | ✅ `delete-campeonato` / `delete-partida` spec |
| **AC-28** | Trocar o dia de uma partida aciona a auditoria em `agendamentos` | HP-07, EX-11, EX-19, EX-20 | ✅ `trocar-dia-partida` spec |

**Todos os 28 AC rastreados a ≥ 1 cenário.** Cenários **só manuais** (sem equivalente automatizado direto): EX-17, EX-18, EX-25, ED-11, ED-13, ED-14 — todos de infraestrutura/integração/limites operacionais, fora do alcance de Jest com infra mockada.

---

## 8. Observações para quem executar

1. **AC com texto legado:** AC-4, AC-5, AC-22 no `plan.md` citam endpoints/fluxos da versão original (`/campeonatos/:id/agendamentos/verificar`, entidade-mãe em `agendamentos`, ADMIN nas rotas de agendamento). A "Revisão arquitetural" do `plan.md` os substituiu — este roteiro testa o **estado construído**. Se `/speckit-documentation` ainda não rodou, os contratos reais estão só no código + neste arquivo.
2. **Ordem sugerida:** RG-09/RG-10 (sobe tudo) → seção 2 (felizes) → seção 3–5 (exceções/edges) → seção 6 (regressão) por último.
3. **Reset entre blocos:** cenários de `cancelar_conflitos:true` (EX-03, EX-04, EX-12) **cancelam** mensalista/aula/ranking reais — reconstruir o seed da §1.3 antes de repetir.
4. **Bug já corrigido (não é achado novo):** EX-04 e RG-10 cobrem a correção feita no `/speckit-validate` (wiring do cancelamento de conflito RANKING no `ConflictResolutionService`).
