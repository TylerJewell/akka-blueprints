# Akka Sample: Hook Instrumentation

A single-agent governance sample that wraps every tool call and LLM invocation with pre/post hook observers. Hooks intercept each activity, enforce validation rules, emit structured log entries, and can inject side effects — all without modifying the agent's core decision logic.

Demonstrates the **single-agent** coordination pattern with three governance controls: a `before-tool-call` guardrail that blocks disallowed tool invocations, an `after-tool-call` guardrail that validates and sanitizes tool outputs before they re-enter the conversation, and a `before-llm-call` guardrail that redacts sensitive context before the prompt leaves the process.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the agent's tool registry and hook chain run in-process.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.akka-hook-instrumentation  ~/my-projects/hook-instrumentation
cd ~/my-projects/hook-instrumentation
```

(Optional) Edit `SPEC.md` to point at a different tool registry (e.g., add a web-search tool or a database-query tool alongside the seeded calculator and file-reader stubs).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ActivityObserverAgent** — an AutonomousAgent that executes tasks using a small set of registered tools, with three hook observers wired around every tool call and LLM invocation.
- **ObservationWorkflow** — orchestrates submit → hook-init → execute → score per submitted task.
- **ObservationEntity** — an EventSourcedEntity holding the per-task lifecycle and the full hook log.
- **HookLogConsumer** — a Consumer that subscribes to `TaskSubmitted` events, initializes the hook chain configuration, and starts the workflow.
- **ObservationView + ObservationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded tool registry for your own (the JSONL file under `src/main/resources/sample-events/tool-registry.jsonl` after generation).
- `SPEC.md §5` — extend `HookLogEntry` with domain-specific fields (e.g., `policyId`, `costEstimate`, `dataClassification`).
- `prompts/activity-observer.md` — narrow the agent's role to a specific task type (a financial agent would constrain tool calls to approved data sources; a code-execution agent would add output-size limits).
- `eval-matrix.yaml` — wire a real policy engine by naming it in the `before-tool-call` guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task → the hook chain initializes → the agent executes → a completed observation with a full hook log appears in the UI.
2. The agent attempts a tool call to a blocked tool name — the `before-tool-call` guardrail rejects it and the agent recovers with an alternative strategy; the rejection is visible in the hook log.
3. A tool returns output containing a sensitive pattern — the `after-tool-call` guardrail redacts it before re-entering the agent's context; the original output is in the entity audit log, not in the agent's conversation.
4. The LLM prompt assembled for a step contains a credential-like token — the `before-llm-call` guardrail strips it; the scrubbed prompt is what the model sees.

## License

Apache 2.0.
