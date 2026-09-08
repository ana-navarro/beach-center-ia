# 001 — Conformidade dos serviços de backend com a Arquitetura Hexagonal

## Título e descrição

Fazer com que os repositórios de backend do ecossistema Beach Center respeitem
**corretamente** a Arquitetura Hexagonal (Ports & Adapters) definida no Princípio II da
constituição. Hoje os serviços têm as pastas `applications/`, `domain/` e `infra/`, mas as
**regras de comunicação e isolamento via ports não são cumpridas**.

Esta é uma tarefa de **refatoração estrutural / conformidade arquitetural** — não adiciona
funcionalidade de negócio. O comportamento externo (endpoints, contratos, respostas) MUST
permanecer idêntico.

## Serviço(s) alvo (Princípio I)

| Repositório | Estado atual | Entra nesta task? |
|---|---|---|
| `services/beach-center-bff-agendamentos` | TS, estrutura completa, **sem `ports/`**, sem testes | **Sim** |
| `services/beach-center-bff-pagamentos` | TS, estrutura completa, **sem `ports/`**, `config` no lugar errado | **Sim** |
| `services/beach-center-bff-usuarios` | TS, estrutura completa, **sem `ports/`** | **Sim** |
| `services/beach-center-whatsapp` | JS (não TS), `domain/` praticamente vazio | **Não** — decisão do usuário: bot ainda não configurado, sem regras definidas |
| `services/beach-center-bff-aulas` | Repositório vazio ("A SER CRIADO" na constituição) | **Não** (fora de escopo) |
| `services/beach-center-bff-campeonatos` | Repositório vazio **e não citado na constituição** | **Não** (fora de escopo; pendência registrada) |

Justificativa: o Princípio II se aplica a "todos os micro-serviços (Node.js/TypeScript)".
Serviços vazios não têm código a refatorar. `whatsapp` fica **fora desta task** por decisão do
usuário — será um bot de WhatsApp ainda não configurado, sem regras de negócio definidas;
entra em task própria quando o escopo estiver claro.

## Diagnóstico — violações do Princípio II encontradas

1. **Camada `ports/` inexistente** em todos os serviços. A constituição exige:
   - `domain/ports/` — interfaces de Input/Output do domínio.
   - `infra/ports/` — interfaces de Output que os usecases chamam para acessar os adapters.
   Nenhum serviço possui essas pastas.

2. **Usecases importam Adapters diretamente** (violação de "domain fala com infra APENAS via
   ports"):
   - `agendamentos`: ~30 imports de `infra/adapters/...` dentro de `domain/usecases/...`
     (court, day, events_scheduled, reserva, scheduling, public-reserve-link).
   - `usuarios`: 7 usecases importam `infra/adapters/...` diretamente.
   - `pagamentos`: `checkout`, `refund` e `webhook` usecases importam `infra/adapters/...`.

3. **Controllers acessam `infra/` diretamente, pulando o domínio** (violação de "applications
   fala com domain APENAS via ports"):
   - `agendamentos`: praticamente todos os controllers importam `infra/adapters` ou
     `infra/firebase`.
   - `pagamentos`: todos os 4 controllers (checkout, refund, transaction-history, webhook).
   - `usuarios`: todos os controllers de `auth/`, além de delete/list/read/update-user.

4. **Regra de negócio dentro de `infra/adapters/`** (adapters "NÃO podem conter regras de
   negócio"):
   - `agendamentos/src/infra/adapters/scheduling/validators/`
   - `agendamentos/src/infra/adapters/events_scheduled/shared/event-conflict` — importado por
     `domain/usecases/reserva/shared/reserve-scheduling.validator.ts` (regra de conflito de
     reserva vivendo em infra).
   - Pastas `types/` e `shared/` espalhadas em `infra/adapters/*` misturando tipos de domínio
     com detalhe de infraestrutura.

5. **`config/` fora do lugar** (constituição: `config/` na raiz do serviço, ex.: `src/config/`):
   - `pagamentos` tem `src/applications/config`.
   - `agendamentos`, `usuarios` não têm pasta `config/` dedicada.

6. **Pastas de `infra/` fora do tripé `adapters | schemas | ports`**:
   - `infra/firebase/` (todos os serviços) e `infra/services/` (`pagamentos`, `whatsapp`) —
     precisam ser reconciliadas: viram `adapters/` específicos + `ports/`, ou `config/`.

7. **Ponto de entrada**: nenhum `main.ts`/`index.ts` em `src/` foi localizado (a constituição
   pede `main.ts`/`index.ts` como root do serviço). O entrypoint atual precisa ser mapeado.

8. **`whatsapp`** (fora do escopo desta task): `domain/` contém apenas
   `default-message-templates.js`; sem `usecases/`, `models/`, `ports/`, `adapters/`. Serviço em
   JS puro, sem `tsconfig.json`. Registrado apenas para referência.

## Regras de negócio / arquiteturais conhecidas (da constituição, Princípio II)

- `domain/usecases/` concentra TODAS as regras de negócio; nunca acessa `schemas`, só `models`.
- `domain` ↔ `infra` **somente** via ports; `applications` ↔ `domain` **somente** via ports.
- `infra/adapters/` sem regra de negócio; **um adapter por verbo/ação**.
- Estrutura de pastas padrão: `applications/{controllers,routes,middlewares,dto}`,
  `domain/{usecases,models,ports}`, `infra/{adapters,schemas,ports}`, `config/`, `main.ts`.
- Comportamento externo e contratos MUST ser preservados (refatoração sem regressão).
- Princípio III: cobertura de testes ≥ 80% para o código tocado; hoje `agendamentos` tem 0
  testes.

## Decisões (respondidas pelo usuário)

1. **Abrangência**: os **3 serviços TS** (`agendamentos`, `pagamentos`, `usuarios`) nesta mesma
   task. O `/speckit-implement` refatora **um serviço por vez**, ordem sugerida
   `usuarios` → `pagamentos` → `agendamentos` (do menor/mais simples ao maior).
2. **`whatsapp`**: **fora** desta task (bot ainda não configurado).
3. **`campeonatos`**: **fora** desta task. Pendência registrada abaixo.
4. **Testes**: **incluídos**. Cada serviço tocado sai da task com **cobertura ≥ 80%** de testes
   unitários `Jest`, mockando completamente as Portas de infra (Princípio III). Inclui criar do
   zero a suíte de `agendamentos` (hoje 0 testes).
5. **Estratégia de ports** (default proposto — confirmar no `/speckit-plan`):
   - `domain/ports/input/` — contrato de cada usecase (o que `applications` consome).
   - `domain/ports/output/` — contrato do que o usecase precisa da infra.
   - `infra/ports/` — reexport/implementação-alvo dos ports de output; cada `infra/adapters/<ação>`
     implementa um port de output.
   - Injeção de dependência **manual** no wiring de cada rota / `config` (factory por ação:
     adapter → port → usecase → controller). Sem container de DI novo.
6. **Preservação de comportamento**: refatoração **sem regressão** — nenhum contrato de endpoint
   muda (paths, payloads, status codes). Correções de bug de contrato ficam para task separada.
7. **`config/` e entrypoint**: padronizar `src/config/` e `src/main.ts` explícito por serviço,
   ajustando `package.json` e scripts do Docker (`beach-center-server`) quando necessário.

## Pendências fora do escopo (registradas)

- `beach-center-bff-campeonatos`: repositório existe mas **não está na constituição** (Princípio
  I). Ação futura: rodar `/speckit-constitution` para registrá-lo, ou removê-lo do ecossistema.
- `beach-center-bff-whatsapp` (`beach-center-whatsapp`): migração JS→TS + camada hexagonal do
  bot, quando o escopo do bot estiver definido.
- `beach-center-bff-aulas`: serviço "A SER CRIADO" — task própria de criação.
- Pastas `documentations/` dentro de `beach-center-bff-agendamentos` e `beach-center-bff-pagamentos`:
  resquício da constituição anterior; a doc agora vive em `beach-center-documentations/`
  (via `/speckit-documentation`).

## Impacto arquitetural previsto (camadas hexagonais)

- **`domain/ports/`** — criar: interfaces de input (contrato de cada usecase) e a referência
  aos ports de output.
- **`infra/ports/`** — criar: uma interface de output por ação (o que hoje é chamado direto do
  adapter).
- **`domain/usecases/`** — alterar: trocar imports de `infra/adapters` por `infra/ports`;
  receber dependências por construtor/parâmetro.
- **`domain/models/`** — possível: mover tipos de domínio que hoje estão em
  `infra/adapters/*/types`.
- **`infra/adapters/`** — alterar: passar a implementar os `infra/ports`; remover regra de
  negócio (`validators`, `event-conflict`) para `domain/usecases`.
- **`infra/schemas/`** — revisar: garantir que só adapters acessam schemas.
- **`applications/controllers/`** — alterar: parar de importar `infra/*`; passar a chamar
  `domain` via port de input.
- **`applications/routes/` + `config/` + `main.ts`** — alterar: wiring/injeção de dependências
  (adapter → port → usecase → controller).
- **Testes (`*.spec.ts`)** — criar: suíte `Jest` por serviço, portas de infra mockadas,
  cobertura ≥ 80%; começar do zero em `agendamentos`.

## Próximo passo

Contexto fechado. Rodar `/speckit-plan` para gerar o checklist de implementação e os Critérios
de Aceite (um bloco por serviço: `usuarios` → `pagamentos` → `agendamentos`).
