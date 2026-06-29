# Akka Sample: LLMCompiler

A Planner compiles the user query into a DAG of tool calls, identifies which calls are independent, and hands them to a Task Fetching Unit that dispatches them in parallel. A Joiner synthesizes all results into a single coherent answer. Demonstrates the **planner-executor** coordination pattern with parallel fan-out and embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** All tool calls — search, calculator, lookup, code-eval — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.llm-compiler-dag  ~/my-projects/llm-compiler-dag
cd ~/my-projects/llm-compiler-dag
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the Planner's tool registry.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that parses the user query and produces a `CompilationPlan`: a list of `ToolCall` nodes with explicit inter-call dependencies forming a DAG.
- **JoinerAgent** — AutonomousAgent that receives all resolved `ToolResult` records and synthesizes a final `QueryAnswer`.
- **TaskFetchingUnit** — Workflow that reads the DAG, identifies the frontier (calls with no unmet dependencies), dispatches them in parallel via concurrent workflow steps, waits for all results, then re-evaluates the frontier until the DAG is exhausted.
- **JobEntity** — EventSourcedEntity holding the job's lifecycle, the compiled DAG, and the accumulated results.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **JobView** — projection used by the UI.
- **JobRequestConsumer** — Consumer that starts a `TaskFetchingUnit` workflow per submission.
- **RequestSimulator** — TimedAction that drips sample queries every 90 s.
- **StaleJobMonitor** — TimedAction that marks jobs stuck in `RUNNING` past 5 minutes as `STALE`.
- **JobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ToolCall` node fields (e.g., add `timeoutOverride`).
- `prompts/planner.md` — narrow which tool types the Planner may emit.
- `eval-matrix.yaml` — add an `eval-event` control for per-result quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → Planner compiles a DAG, Task Fetching Unit dispatches independent calls in parallel, Joiner produces an answer within ~3 minutes.
2. Inject a tool call to a non-allow-listed tool type → guardrail blocks it; Planner revises the DAG; job completes via a different path or fails within budget.
3. Submit a query and click **Halt new dispatches** while the job is `RUNNING` — in-flight parallel calls finish; no new frontier is dispatched; job moves to `HALTED`.
4. A tool result containing a secret-shaped string is scrubbed before it reaches the Joiner's synthesis prompt.

## License

Apache 2.0.
