# run-workflow skill

**English** | [Português](README.pt-BR.md)

A copy of the `run-workflow` skill (namespace `system`) that showed up in my project after I installed the skill for Junie, JetBrains' AI coding agent, in IntelliJ. The original content is in [`SKILL.md`](SKILL.md), unchanged.

## What the skill teaches

It teaches the agent to use the `run_workflow` tool. Instead of calling subagents one at a time, the agent writes a **short Python script that orchestrates many agents**. The script decides which agents to start, in what order, and what to do with each result.

### When to use it

- The task splits into many independent or staged pieces, such as analyzing several files or modules, or reviewing from several angles, and then aggregating the results.
- The user asks for a workflow.

### When not to use it

- For a single delegated task (a subagent is enough).
- For legacy workflow files (a separate skill, `migrate-workflows`, covers those).

## How it works

1. The agent passes the script inline (`Script`) or as a file (`ScriptPath`), along with a one-line `Description`.
2. The workflow runs in the background, and the agent is notified when it ends. Inputs go in `Args`, which the script reads as the `args` dict.
3. The result contains the script's `log()` lines (`Logs:`) and whatever it printed (`Output:`).
4. On an error, the agent fixes the reported line and runs it again.

## Primitives

| Primitive | What it does |
| --- | --- |
| `agent(prompt, schema, role, type_name, workspace, model)` | Defines an agent. `await agent(...)` runs it once and returns the result. Calling `a(prompt, schema)` keeps a conversation with context across turns. |
| `parallel(items, fn)` | Runs `fn(item)` for every item concurrently. If one fails, the others are cancelled. |
| `pipeline(items, fn)` | Runs `fn(item)` one item at a time, in order. |
| `phase(name)` | Groups the following logs and agents under a named stage. |
| `log(*args)` | Writes a line to the workflow output and the UI. |
| `call_tool(name, **args)` | Calls a tool directly from the script and returns its output as a `str`. |
| `trajectory(...)` | Reads earlier steps of the conversation, such as the output of a failed command. |

## Key points

- **Schema-checked output:** every agent needs a `schema` (JSON schema) and returns its result by calling the `finish` tool. An agent that ends without calling `finish` fails the workflow.
- **Workspace:** `"inherit"` uses the current directory, `"branch"` creates an isolated clone (for agents that edit code in parallel), and `"share"` creates a workspace that shares the repository.
- **Model:** `"inherit"`, `"flash_lite"`, `"flash"`, or `"pro"`. Lighter tiers suit simple work fanned out to many agents.
- **Context:** agents see neither the script nor each other. All the context has to go in the prompt, with a request for a short, well-defined output.
- **Lazy execution:** an `agent()` that is never awaited never runs.
- **No user interaction:** a workflow cannot ask the user anything. A tool call that would need approval is denied.

## Example

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

One agent lists the files, then each file gets its own reviewer running in parallel.
