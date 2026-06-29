# Akka Sample: Shared Task Team

A lead agent receives a goal, breaks it into tasks on a shared list, and a team of worker agents claim tasks on their own, produce outputs, run a quality check, and message each other when they need to coordinate. Demonstrates the **team-shared-list** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the task workspace, the quality checker, and the goal intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.general.shared-task-team  ~/my-projects/shared-task-team
cd ~/my-projects/shared-task-team
```

(Optional) Edit `SPEC.md` to change the system name, the number of worker agents, the model provider, or the goal briefs the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GoalPlanner** — AutonomousAgent that decomposes a goal brief into a list of tasks with dependencies.
- **WorkerAgent** — AutonomousAgent (run as several worker instances) that claims a task, produces a result artifact, and raises a peer request when it needs another worker's output.
- **PlanningWorkflow** — Workflow that runs the goal planner and writes one task per plan onto the board.
- **WorkerWorkflow** — one durable loop per worker: poll the board, claim an open task atomically, produce an artifact under a tool guardrail, pass the quality check, and mark the task done or blocked.
- **TaskEntity** — one EventSourcedEntity per task; the atomic claim that stops two workers grabbing the same task.
- **GoalEntity**, **WorkerMailbox**, **IntakeQueue** — goal lifecycle, per-worker peer messages, and the submission log.
- **SystemControl** — KeyValueEntity holding the operator halt flag.
- **TaskBoardView** — the shared task list the UI and the workers read.
- **SharedTaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the goal briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the worker roster (`worker-1`, `worker-2`, `worker-3`) to a different count.
- `prompts/worker-agent.md` — narrow the worker to a specific output domain.
- `eval-matrix.yaml` — extend the before-agent-response guardrail's refusal criteria for your domain.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a goal → the planner writes tasks to the board → workers claim and complete them → the goal reaches `COMPLETED`.
2. Two workers claim concurrently → each task is claimed by exactly one worker; no double-claim.
3. A worker raises a peer request → the task goes `BLOCKED`, a message lands in the peer's mailbox, the reply unblocks it.
4. The planner produces output that fails the safety guardrail → the guardrail blocks the response before it reaches the user; the task is recorded `BLOCKED` with the guardrail reason.
5. The operator halts the team → in-flight claiming pauses and response generation is refused; resume continues the work.

## License

Apache 2.0.
