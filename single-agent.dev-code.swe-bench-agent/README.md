# Akka Sample: SWE-Bench Agent

A single software-engineering agent receives a task description and a repository snapshot, applies a patch, and reports a structured outcome: PASS / FAIL / NEEDS_RETRY with a per-test breakdown. The repository snapshot is delivered to the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a CI gate that runs the repository's test suite against every candidate patch before the result is accepted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the task corpus lives in-process and the agent's test-runner is simulated.

## Generate the system

```sh
cp -r ./single-agent.dev-code.swe-bench-agent  ~/my-projects/swe-bench-agent
cd ~/my-projects/swe-bench-agent
```

(Optional) Edit `SPEC.md` to point at a different task corpus (e.g., swap the seeded Python issues for JavaScript or Go repository snapshots).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PatchEngineerAgent** — an AutonomousAgent that accepts a bug description and a repository snapshot attachment, produces a structured `PatchResult` with the generated diff and a confidence score.
- **BenchmarkWorkflow** — orchestrates prepare-wait → patch → test-gate per submitted task.
- **BenchmarkTaskEntity** — an EventSourcedEntity holding the per-task lifecycle.
- **SnapshotPreparer** — a Consumer that subscribes to `TaskSubmitted` events, normalizes the repository snapshot into a `PreparedSnapshot`, and emits `SnapshotPrepared` back to the entity.
- **BenchmarkView + BenchmarkEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customize before generating

- `SPEC.md §3` — swap the seeded task corpus for your own repository snapshots (the JSONL file under `src/main/resources/sample-events/seed-tasks.jsonl` after generation).
- `SPEC.md §5` — extend `PatchResult` with project-specific fields (e.g., `coverageDelta`, `lintScore`, `affectedModules`).
- `prompts/patch-engineer.md` — narrow the agent's role (a Go project would constrain it to module-aware patching conventions; a TypeScript project to strict type-checking rules).
- `eval-matrix.yaml` — wire a real test runner (e.g., a sandboxed subprocess executor) by naming it under the ci-gate mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a task + snapshot → it is prepared → the agent patches → the test gate runs → the result appears in the UI.
2. The agent produces a patch that fails the test gate → the gate blocks the result → the agent retries → a passing patch lands.
3. Every recorded result has a CI gate report visible on the same UI card.
4. Repository snapshots submitted with injected secrets never appear in the LLM call log; only the sanitized form does.

## License

Apache 2.0.
