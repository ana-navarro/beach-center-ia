# Beach Center — Documentação dos Endpoints

Documentação das APIs de todos os microsserviços de backend do ecossistema Beach Center,
em português, um arquivo por rota lógica.

Gerada e mantida pelo comando `/speckit-documentation` (passo 8 do Ciclo Speckt), a partir do
**código atual** de cada repositório (`applications/routes/` + `controllers/` + `dto/` +
`domain/usecases/`).

## Estrutura

```
beach-center-documentation/
  <nome-do-repo>/
    <nome-da-rota>.md
```

| Serviço | Rotas documentadas |
|---|---|
| `beach-center-bff-agendamentos` | `agendamento`, `aula-bloqueio`, `campeonato-agendamento`, `dia`, `evento-agendado`, `link-reserva-publica`, `mensalista`, `mensalista-plano`, `quadra`, `ranking-agendamento`, `reserva`, `unidade` |
| `beach-center-bff-aulas` | `aula`, `aluno` |
| `beach-center-bff-campeonatos` | `campeonato`, `partida`, `ranking` |
| `beach-center-bff-pagamentos` | `checkout`, `comprovante-pagamento`, `forma-pagamento`, `webhook-getnet` |
| `beach-center-bff-usuarios` | `auth`, `usuario` |
| `beach-center-whatsapp` | `mensagem`, `modelo-mensagem`, `webhook` |

## Estrutura de cada documento

- **Visão geral** — o que a rota representa e qual serviço a expõe.
- **Autenticação/autorização** — guards e perfis exigidos.
- **Endpoints** — por método+path: descrição/regra de negócio (usecase), parâmetros e corpo,
  resposta de sucesso (exemplo + código HTTP), erros possíveis, exemplo de chamada (`curl`).
- **Referências** — caminhos dos arquivos de rota/controller/usecase no código.
