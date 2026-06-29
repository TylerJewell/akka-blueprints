# Akka Sample: Dev Team Task Board

A team lead agent splits a software project into tasks on a shared board; developer agents claim tasks on their own, write code, run a test gate, and message each other when they need to coordinate. Demonstrates the **team-shared-list** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the code workspace, the test runner, and the project intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.dev-code.devteam  ~/my-projects/dev-team-task-board
cd ~/my-projects/dev-team-task-board
```

(Optional) Edit `SPEC.md` to change the system name, the number of developer agents, the model provider, or the project briefs the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TeamLead** — AutonomousAgent that decomposes a project brief into a list of tasks with dependencies.
- **DeveloperAgent** — AutonomousAgent (run as several developer instances) that claims a task, writes the code artifact, and raises a peer request when it needs another developer's output.
- **PlanningWorkflow** — Workflow that runs the team lead and writes one task per spec onto the board.
- **DeveloperWorkflow** — one durable loop per developer: poll the board, claim an open task atomically, write code under a tool guardrail, pass the test gate, and mark the task done or blocked.
- **TaskEntity** — one EventSourcedEntity per task; the atomic claim that stops two developers grabbing the same task.
- **ProjectEntity**, **PeerMailbox**, **IntakeQueue** — project lifecycle, per-developer peer messages, and the submission log.
- **SystemControl** — KeyValueEntity holding the operator halt flag.
- **TaskBoardView** — the shared task list the UI and the developers read.
- **DevTeamEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the project briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the developer-agent roster (`dev-1`, `dev-2`, `dev-3`) to a different count.
- `prompts/developer.md` — narrow the developer to a single language or stack.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's destructive-operation list if you wire a real shell or file tool.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a project → the team lead splits it into tasks → developers claim and complete them → the project reaches `COMPLETED`.
2. Two developers claim concurrently → each task is claimed by exactly one developer; no double-claim.
3. A developer raises a peer request → the task goes `BLOCKED`, a message lands in the peer's mailbox, the reply unblocks it.
4. A developer attempts a destructive tool call → the before-tool-call guardrail blocks it before execution; the task is recorded `BLOCKED` with the guardrail reason.
5. The operator halts the team → in-flight claiming pauses and tool calls are refused; resume continues the work.

## License

Apache 2.0.
