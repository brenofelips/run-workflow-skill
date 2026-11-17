# run-workflow skill

Cópia da skill `run-workflow` (namespace `system`) que apareceu no meu projeto depois de instalar a skill do Junie, o agente de IA da JetBrains, no IntelliJ. O conteúdo original está em [`SKILL.md`](SKILL.md), sem alterações.

## O que a skill orienta

Ela ensina o agente a usar a ferramenta `run_workflow`: em vez de chamar subagentes um por um, o agente escreve um **script Python curto que orquestra vários agentes**. O script decide quais agentes iniciar, em que ordem, e o que fazer com o resultado de cada um.

### Quando usar

- A tarefa se divide em muitos pedaços independentes ou em etapas, como analisar vários arquivos ou módulos, revisar por vários ângulos e depois juntar tudo.
- O usuário pede um workflow.

### Quando não usar

- Para uma única tarefa delegada (nesse caso, um subagente basta).
- Para arquivos de workflow legados (para isso existe outra skill, a `migrate-workflows`).

## Como funciona

1. O agente passa o script inline (`Script`) ou como arquivo (`ScriptPath`), junto com uma `Description` de uma linha.
2. O workflow roda em segundo plano e o agente é avisado quando termina. Entradas vão em `Args`, que o script lê como o dict `args`.
3. O resultado traz os `log()` do script (`Logs:`) e o que ele imprimiu (`Output:`).
4. Se der erro, o agente corrige a linha indicada e roda de novo.

## Primitivas

| Primitiva | O que faz |
| --- | --- |
| `agent(prompt, schema, role, type_name, workspace, model)` | Define um agente. `await agent(...)` roda uma vez e devolve o resultado. Chamar `a(prompt, schema)` mantém uma conversa com contexto entre os turnos. |
| `parallel(items, fn)` | Roda `fn(item)` para todos os itens ao mesmo tempo. Se um falha, os outros são cancelados. |
| `pipeline(items, fn)` | Roda `fn(item)` um item por vez, em ordem. |
| `phase(name)` | Agrupa os logs e agentes seguintes sob uma etapa nomeada. |
| `log(*args)` | Escreve uma linha na saída do workflow e na interface. |
| `call_tool(name, **args)` | Chama uma ferramenta direto do script e devolve a saída como `str`. |
| `trajectory(...)` | Lê os passos anteriores da conversa, como a saída de um comando que falhou. |

## Pontos importantes

- **Saída com schema:** todo agente precisa de um `schema` (JSON schema) e devolve o resultado chamando a ferramenta `finish`. Se terminar sem chamar `finish`, o workflow falha.
- **Workspace:** `"inherit"` usa o diretório atual, `"branch"` cria um clone isolado (para agentes que editam código em paralelo) e `"share"` cria um workspace que compartilha o repositório.
- **Modelo:** `"inherit"`, `"flash_lite"`, `"flash"` ou `"pro"`. Tiers mais leves servem para tarefas simples com muitos agentes em paralelo.
- **Contexto:** os agentes não veem o script nem uns aos outros. Todo o contexto precisa ir no prompt, pedindo uma saída curta e bem definida.
- **Execução preguiçosa:** um `agent()` que nunca é aguardado (`await`) nunca roda.
- **Sem interação:** o workflow não consegue perguntar nada ao usuário. Uma chamada de ferramenta que precisaria de aprovação é negada.

## Exemplo

```python
phase("Discover")
files = await agent(
    "List the Go files under pkg/foo that call Bar(). Paths only.",
    schema={"type": "array", "items": {"type": "string"}},
)

phase("Review")
reviews = await parallel(files, lambda f: agent(
    f"Review {f} for misuse of Bar(). Reply in 3 bullets or 'OK'.",
    schema={"type": "string"},
    role=f"Review {f}",
))
for r in reviews:
    log(r)
```

Um agente lista os arquivos e, em seguida, cada arquivo ganha um revisor rodando em paralelo.
