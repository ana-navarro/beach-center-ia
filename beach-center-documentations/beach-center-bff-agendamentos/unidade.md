# Unidade

## Visão geral

Recurso `unit`, exposto pelo serviço **beach-center-bff-agendamentos** na rota base `/api/v1/unidades`. Representa uma unidade física do Beach Center (endereço + conjunto de quadras vinculadas). É consultado por outros recursos do mesmo serviço (ex.: `scheduling`, `day`) para validar se uma quadra pertence a uma unidade e para resolver o horário de funcionamento usado na criação de agendamentos/dias — essa resolução de horário de funcionamento (incluindo o caso especial da unidade `6a440a931094fad2f585011b`, com janela 18:00–22:00 sem almoço) não faz parte do código deste recurso; é documentada em `agendamento.md` e `dia.md`.

## Autenticação/autorização

- `GET /unidades/:id` e `GET /unidades`: públicos, sem autenticação.
- `POST /unidades`, `PATCH /unidades/:id`, `PATCH /unidades/:id/delete`: exigem `Authorization: Bearer <idToken>` válido (`authMiddleware`) **e** `requireRole('ADMIN')` — usuário autenticado precisa ter `user_type: "ADMIN"` na coleção `users` espelhada.

## Endpoints

### GET /api/v1/unidades/:id

Busca uma unidade pelo `id`. Aciona `ReadUnitUsecase`, que valida o formato do `id` (ObjectId de 24 caracteres hex) e lança erro de domínio se a unidade não existir ou estiver com `deleted: true` (o adapter de leitura não retorna registros deletados).

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId, 24 hex) | Sim | Identificador da unidade |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Unidade encontrada com sucesso",
  "data": {
    "id": "6a440a931094fad2f585011b",
    "name": "Praia Náutica",
    "location": {
      "street": "Av. Beira Mar",
      "quarter": "Centro",
      "number": "100",
      "city": "Florianópolis",
      "state": "SC",
      "cep": "88000-000"
    },
    "courts": ["651f0a1b2c3d4e5f60718293"],
    "deleted": false
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é um ObjectId válido de 24 caracteres hex — `message: "ID inválido"` |
| 404 | Unidade não encontrada (inexistente ou soft-deleted) — `message: "Unidade não encontrada"` |
| 500 | Erro interno inesperado |

**Exemplo de chamada**

```bash
curl -X GET "https://<host>/api/v1/unidades/6a440a931094fad2f585011b"
```

---

### GET /api/v1/unidades

Lista todas as unidades não deletadas. Aciona `ListUnitsUsecase`. Sem filtro, retorna a lista completa. Com `?search=` (ou `?name=`, alias legado — `search` tem prioridade se ambos forem enviados), aplica busca textual **normalizada** (remove acentos via NFD, remove pontuação, colapsa espaços, lowercase) sobre a concatenação de `name` + todos os campos de `location` (`street`, `number`, `quarter`, `city`, `state`, `cep`); o termo de busca é dividido em palavras e **todas** precisam aparecer no conteúdo normalizado (AND, não OR). Se o termo for vazio ou só espaços após `trim()`, o filtro é ignorado e todas as unidades são retornadas.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `search` | string | Não | Busca textual insensível a acento/caixa sobre nome e endereço (todas as palavras devem casar) |
| `name` | string | Não | Alias legado de `search` — usado apenas se `search` não for enviado |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Unidades listadas com sucesso",
  "data": [
    {
      "id": "6a440a931094fad2f585011b",
      "name": "Praia Náutica",
      "location": { "street": "Av. Beira Mar", "quarter": "Centro", "number": "100", "city": "Florianópolis", "state": "SC", "cep": "88000-000" },
      "courts": ["651f0a1b2c3d4e5f60718293"],
      "deleted": false
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 500 | Erro interno inesperado |

**Exemplo de chamada**

```bash
curl -X GET "https://<host>/api/v1/unidades?search=nautica"
```

---

### POST /api/v1/unidades

Cria uma nova unidade. Aciona `CreateUnitUsecase`, que persiste a unidade com `deleted: false`. Não valida no usecase se os `courts` informados existem de fato (a validação de formato de ObjectId é feita no DTO).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome da unidade |
| `location.street` | string | Sim | Logradouro |
| `location.quarter` | string | Sim | Bairro |
| `location.number` | string | Sim | Número |
| `location.city` | string | Sim | Cidade |
| `location.state` | string | Sim | UF |
| `location.cep` | string | Sim | CEP |
| `courts` | string[] | Sim | Lista de `id`s de quadras (ObjectId de 24 caracteres hex cada) |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Unit created successfully",
  "data": {
    "id": "6a440a931094fad2f585011b",
    "name": "Praia Náutica",
    "location": { "street": "Av. Beira Mar", "quarter": "Centro", "number": "100", "city": "Florianópolis", "state": "SC", "cep": "88000-000" },
    "courts": ["651f0a1b2c3d4e5f60718293"],
    "deleted": false
  }
}
```

> Nota: a mensagem de sucesso é em **inglês** (`"Unit created successfully"`) — inconsistência pré-existente em relação ao restante do serviço (PT-BR), preservada deliberadamente (não é regressão a corrigir).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido (campo obrigatório ausente, ou item de `courts` que não bate no regex de ObjectId de 24 hex) — `message: "Dados inválidos"`, `errors: [{field, message}]` |
| 401 | Sem token / token inválido |
| 403 | Usuário autenticado não é `ADMIN` |
| 500 | Erro interno inesperado |

**Exemplo de chamada**

```bash
curl -X POST "https://<host>/api/v1/unidades" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Praia Náutica",
    "location": {"street": "Av. Beira Mar", "quarter": "Centro", "number": "100", "city": "Florianópolis", "state": "SC", "cep": "88000-000"},
    "courts": ["651f0a1b2c3d4e5f60718293"]
  }'
```

---

### PATCH /api/v1/unidades/:id

Atualiza uma unidade existente. Aciona `UpdateUnitUsecase`, que sobrescreve todos os campos (não é um merge parcial — todo o payload do DTO é obrigatório, igual ao `create`). Retorna a unidade atualizada (`data`) e uma lista (`list`) de todas as unidades não deletadas.

> ⚠️ Comportamento pré-existente preservado: em `list`, cada item retornado usa os campos de `location`/`name`/`courts`/`deleted` do **payload de entrada** desta requisição (não os dados reais de cada unidade da coleção) — apenas o `id` de cada item é o real. Não corrigir; é um comportamento documentado do código original, não introduzido por refatoração.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId, 24 hex) | Sim | Identificador da unidade |

**Corpo da requisição**

Mesmo shape do `POST /unidades` (todos os campos obrigatórios: `name`, `location.*`, `courts`).

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Unidade encontrada com sucesso",
  "data": {
    "id": "6a440a931094fad2f585011b",
    "name": "Praia Náutica renomeada",
    "location": { "street": "Av. Beira Mar", "quarter": "Centro", "number": "100", "city": "Florianópolis", "state": "SC", "cep": "88000-000" },
    "courts": ["651f0a1b2c3d4e5f60718293"],
    "deleted": false
  },
  "list": [ "..." ]
}
```

> Nota: a mensagem de sucesso (`"Unidade encontrada com sucesso"`) é reaproveitada do fluxo de leitura — comportamento pré-existente preservado, não corrigir.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido, ou `id` não é ObjectId válido — `message: "Dados inválidos"` ou `"ID inválido"` |
| 401 | Sem token / token inválido |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | Unidade não encontrada — `message: "Unidade não encontrada"` |
| 500 | Erro interno inesperado |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/unidades/6a440a931094fad2f585011b" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Praia Náutica renomeada",
    "location": {"street": "Av. Beira Mar", "quarter": "Centro", "number": "100", "city": "Florianópolis", "state": "SC", "cep": "88000-000"},
    "courts": ["651f0a1b2c3d4e5f60718293"]
  }'
```

---

### PATCH /api/v1/unidades/:id/delete

Remove (soft-delete) uma unidade. Aciona `DeleteUnitUsecase`, que marca a unidade como deletada e retorna a lista atualizada de unidades não deletadas.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId, 24 hex) | Sim | Identificador da unidade |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Unidade deletada com sucesso",
  "data": [
    { "id": "651f...", "name": "Outra unidade", "location": { "...": "..." }, "courts": ["..."], "deleted": false }
  ]
}
```

> Nota: `data` aqui é a **lista** de unidades remanescentes (não deletadas), não a unidade excluída — comportamento do usecase (`DeleteUnitUsecase` retorna `listUnitsPort.execute()` após deletar).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é ObjectId válido — `message: "ID inválido"` |
| 401 | Sem token / token inválido |
| 403 | Usuário autenticado não é `ADMIN` |
| 404 | Unidade não encontrada — `message: "Unidade não encontrada"` |
| 500 | Erro interno inesperado |

**Exemplo de chamada**

```bash
curl -X PATCH "https://<host>/api/v1/unidades/6a440a931094fad2f585011b/delete" \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `services/beach-center-bff-agendamentos/src/applications/routes/unit.route.ts`
- Controllers: `services/beach-center-bff-agendamentos/src/applications/controllers/unit/{create,read,update,delete,list}/*.controller.ts`
- Usecases: `services/beach-center-bff-agendamentos/src/domain/usecases/unit/{create,read,update,delete,list}/*.usecase.ts`
- DTO: `services/beach-center-bff-agendamentos/src/applications/dto/unit.dto.ts`
- Modelos: `services/beach-center-bff-agendamentos/src/domain/models/unit.model.ts`, `location.model.ts`
- Ports: `services/beach-center-bff-agendamentos/src/domain/ports/input/unit.input-port.ts`, `domain/ports/output/unit-persistence.port.ts`
- Adapters: `services/beach-center-bff-agendamentos/src/infra/adapters/unit/**/*.adapter.ts`
