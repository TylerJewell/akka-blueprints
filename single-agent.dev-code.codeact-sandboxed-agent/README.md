# Akka Sample: CodeAct Agent

A single code-writing agent receives a natural-language task, generates executable code, runs it inside a policy-checked sandbox, inspects the output, and iterates until the task is solved or the iteration budget is exhausted. The agent never executes code without passing a pre-execution guardrail, and an automatic safety halt stops execution the moment destructive or secret-leaking behaviour is detected.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that inspects every `execute_code` call before it runs, an automatic safety halt that interrupts execution mid-run if forbidden patterns emerge, and a secret sanitizer that scrubs credentials and tokens from generated code and from execution outputs before they propagate to the event log or the UI.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — code execution is sandboxed in-process via a restricted Java `ScriptEngine`; no Docker, no external runtime is required.

## Generate the system

```sh
cp -r ./single-agent.dev-code.codeact-sandboxed-agent  ~/my-projects/codeact-agent
cd ~/my-projects/codeact-agent
```

(Optional) Edit `SPEC.md` to change the seeded task catalogue (e.g., switch from data-transformation tasks to API-scripting tasks or algorithmic puzzles) or to tighten the sandbox policy (e.g., disallow file I/O, restrict allowed packages).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeActAgent** — an AutonomousAgent that accepts a natural-language task definition, generates Python-like pseudocode (executed via the in-process sandbox), inspects the output, and iterates until solved or until its iteration budget is exhausted.
- **ExecutionWorkflow** — orchestrates validate → execute → inspect per submitted task, with per-step timeouts and a failover to the error state.
- **TaskEntity** — an EventSourcedEntity holding the per-task lifecycle from `SUBMITTED` through `SOLVED` or `FAILED`.
- **SandboxExecutor** — a pure service class invoked by the workflow; runs generated code through the `before-tool-call` guardrail and the safety halt, then records the `CodeExecuted` event.
- **SecretSanitizer** — a Consumer that subscribes to `CodeExecuted` events, scrubs secrets from outputs, and emits `OutputSanitized` back to the entity.
- **TaskView + TaskEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded task catalogue for your own domain (the JSONL file under `src/main/resources/sample-events/seed-tasks.jsonl` after generation).
- `SPEC.md §5` — extend `ExecutionResult` with fields such as `exitCode`, `memoryUsedBytes`, or `executionTimeMs` if the sandbox can measure them.
- `prompts/codeact-agent.md` — narrow the agent's code-generation style (a data-engineering deployer would constrain it to Polars/pandas transforms; a scripting deployer would constrain it to bash-equivalent pseudocode).
- `eval-matrix.yaml` — wire a real sandboxed runtime (e.g., a gVisor-backed container) by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a data-transformation task → the agent generates code → the sandbox runs it → the output appears in the UI with status `SOLVED`.
2. The agent generates code that contains a forbidden syscall → the `before-tool-call` guardrail rejects it → the agent rewrites the code → a clean version runs successfully.
3. An execution output containing `AWS_SECRET_ACCESS_KEY=AKIA...` is scrubbed by the secret sanitizer before the output lands in the entity log; the UI displays `[REDACTED-SECRET]`.
4. A runaway loop triggers the safety halt → the task transitions to `HALTED` → the UI flags the card with the halt reason.

## License

Apache 2.0.
