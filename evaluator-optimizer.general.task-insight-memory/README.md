# Akka Sample: Task-Centric Memory

An agent accepts tasks, executes them, and stores what it learned as typed insights keyed to the task type. On subsequent tasks of the same kind the agent retrieves relevant prior insights before acting. The memory accumulates through three write paths: corrections from operators, demonstrations submitted by administrators, and verified self-experience where the agent's own outcome is evaluated before the insight is persisted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; all task sources and the memory store are modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.task-insight-memory  ~/my-projects/task-insight-memory
cd ~/my-projects/task-insight-memory
```

(Optional) Edit `SPEC.md` to change the task types the simulator drips, the insight confidence threshold, or the memory retention policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExecutorAgent** — AutonomousAgent that retrieves relevant insights for a task type, executes the task, and returns a typed `TaskResult` with a confidence score.
- **EvaluatorAgent** — AutonomousAgent that scores a completed task result against the original task's acceptance criteria, returns either `VERIFIED` or `REJECTED` with a typed `EvalNotes` payload.
- **MemoryWorkflow** — Workflow that runs the retrieve → execute → evaluate → persist loop; blocked or low-confidence results do not reach the memory store.
- **MemoryEntity** — EventSourcedEntity holding the memory store: all insight records keyed by task type, their provenance (correction / demonstration / verified-experience), and their scrubbed text.
- **TaskEntity** — EventSourcedEntity holding per-task state, every attempt's result, every eval verdict, and the final outcome.
- **MemoryView** — read-side projection of the insight store the UI and the agent query.
- **TaskView** — read-side projection of all tasks, streamed via SSE.
- **TaskConsumer** — Consumer that subscribes to `TaskQueue` events and starts a `MemoryWorkflow` per submitted task.
- **TaskQueue** — EventSourcedEntity that logs each task submission for replay and audit.
- **TaskSimulator** — TimedAction that drips a sample task every 60 s so the App UI is not empty when first loaded.
- **DriftWatcher** — TimedAction that scans the memory store every 5 minutes and emits a `DriftAssessmentRecorded` event when insight distribution deviates from baseline.
- **MemoryEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task types the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `MemoryInsight` record fields (e.g., add domain tags, raise the confidence threshold).
- `prompts/executor.md` — change how the agent selects and weighs retrieved insights.
- `prompts/evaluator.md` — tighten or relax the acceptance criteria applied to results.
- `eval-matrix.yaml` — tighten PII enforcement (currently non-blocking for the sanitizer run).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → result progresses `PENDING` → `EXECUTING` → `VERIFIED`; insight is persisted in the memory store.
2. Submit the same task type again → the agent retrieves the prior insight and the result quality is measurably higher (or the mock provider surfaces the retrieval path).
3. A correction submitted by an operator overwrites a previous insight; the next task execution retrieves the corrected version.
4. The DriftWatcher fires after several cycles and emits a `DriftAssessmentRecorded` event visible in the App UI's memory timeline.

## License

Apache 2.0.
