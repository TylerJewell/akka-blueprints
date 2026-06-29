# Akka Sample: Code Assistant

A single code-assistant agent reads a repository snapshot, runs analysis tools, and proposes a structured edit plan: file changes with before/after diffs, a test command, and a confidence rating. The repository content rides into the agent as task attachments, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that intercepts shell and file-mutation tool calls before the agent executes them, and a CI gate that runs the project's test suite against every proposed edit before the result is surfaced to the developer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the repository corpus lives in-process and the agent's tool calls are sandboxed within the JVM.

## Generate the system

```sh
cp -r ./single-agent.dev-code.code-assistant  ~/my-projects/code-assistant
cd ~/my-projects/code-assistant
```

(Optional) Edit `SPEC.md` to point at a different repository snapshot or to change the set of allowed analysis tools the agent may call.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CodeAssistantAgent** — an AutonomousAgent that accepts a coding task description plus repository file attachments and returns a typed `EditPlan`.
- **EditWorkflow** — orchestrates attach-files → analyze → gate per submitted coding task.
- **EditEntity** — an EventSourcedEntity holding the per-task lifecycle.
- **ToolCallGuardrail** — intercepts shell and file-mutation tool calls before execution; blocks disallowed commands.
- **CIGate** — a Consumer that subscribes to `EditProposed` events, runs the test suite against the proposed changes, and emits `GatePassed` or `GateFailed` back to the entity.
- **EditView + EditEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded repository snapshot for your own (the JSONL file under `src/main/resources/sample-events/repo-snapshots.jsonl` after generation).
- `SPEC.md §5` — extend `EditPlan` with project-specific fields (e.g., `affectedServices`, `breakingChange`, `migrationScript`).
- `prompts/code-assistant.md` — narrow the agent's role (a backend team would restrict it to Java service changes; a platform team might allow infrastructure-as-code edits).
- `eval-matrix.yaml` — wire a real CI runner (e.g., a GitHub Actions webhook) by naming it under the ci-gate mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a coding task + repository snapshot → the agent proposes an edit plan → the CI gate runs → the plan appears in the UI with a gate-passed badge.
2. The agent attempts a disallowed shell command during analysis → the `before-tool-call` guardrail blocks it → the agent retries without the disallowed call → a valid plan lands.
3. An edit plan whose proposed changes break the test suite → the CI gate fails → the card shows a gate-failed badge and the test failure output → the developer is not handed a broken edit.
4. Repository file contents submitted in the task are never embedded in the agent's instruction text; only the file attachments carry the source.

## License

Apache 2.0.
