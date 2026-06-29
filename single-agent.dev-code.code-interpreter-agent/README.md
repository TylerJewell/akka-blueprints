# Akka Sample: Code Interpreter Agent

A single agent receives a user prompt and a data payload, then executes Python code against that data inside a sandboxed interpreter to produce a structured result — a table, a computed value, or a chart description. Two governance mechanisms sit around the one code-running agent: a `before-tool-call` guardrail that vets the generated code before execution, and an automatic safety halt that terminates runaway executions before they exhaust resources.

Demonstrates the **single-agent** coordination pattern wired with two controls that address the specific risks of code generation: pre-execution code inspection and bounded execution enforcement.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The sandboxed interpreter runs in-process; the agent's code-execution tool is simulated at scaffold time.

## Generate the system

```sh
cp -r ./single-agent.dev-code.code-interpreter-agent  ~/my-projects/code-interpreter-agent
cd ~/my-projects/code-interpreter-agent
```

(Optional) Edit `SPEC.md` to restrict which Python modules the agent may import, or to swap in a real interpreter integration (e.g., a subprocess-isolated Python runtime) in place of the in-process simulation.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeInterpreterAgent** — an AutonomousAgent that receives a user prompt and an optional data attachment, generates Python code as a tool call, and returns a typed `ExecutionResult`.
- **InterpretationWorkflow** — orchestrates guard-wait → execute → record per submitted job.
- **JobEntity** — an EventSourcedEntity holding the per-job lifecycle from submitted to result-recorded.
- **ExecutionSandbox** — the in-process code-execution service invoked by the workflow; enforces wall-clock and memory budgets.
- **JobView + JobEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded dataset examples (CSV sample, JSON metrics payload, numeric array) for your own test inputs.
- `SPEC.md §5` — extend `ExecutionResult` with domain-specific output types (e.g., a `ChartSpec` record for visualization outputs).
- `prompts/code-interpreter.md` — tighten the agent's allowed imports list or the maximum number of lines of code it may emit.
- `eval-matrix.yaml` — replace the in-process sandbox with a real isolated runtime by naming it under the halt mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt with a CSV attachment → the agent generates code → the guardrail approves it → execution returns a result visible in the UI.
2. The agent generates code that imports a forbidden module → the `before-tool-call` guardrail rejects it → the agent retries → the approved code runs successfully.
3. A job whose generated code enters an infinite loop is terminated by the safety halt within the configured wall-clock budget; the job transitions to `HALTED` and the UI reflects that.
4. A job submitting a 10,000-row dataset produces the correct aggregated answer and no intermediate rows appear in the LLM call log.

## License

Apache 2.0.
