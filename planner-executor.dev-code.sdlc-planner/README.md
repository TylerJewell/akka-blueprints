# Akka Sample: SDLC Task Planner

A Planner decomposes a software development lifecycle request into discrete tasks, dispatches each one to specialist agents — Analyst, Architect, Coder, Reviewer — tracks outcomes on a task ledger plus a progress ledger, and replans when a step fails. Demonstrates the **planner-executor** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The four specialist surfaces — analysis, architecture, code, review — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.dev-code.sdlc-planner  ~/my-projects/sdlc-task-planner
cd ~/my-projects/sdlc-task-planner
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that maintains a task ledger (requirements, constraints, plan, current dispatch) and a progress ledger (per-task attempts, verdicts, blockers). Decides which specialist runs each step. Replans on three consecutive failures.
- **AnalystAgent** — AutonomousAgent that analyses requirements and acceptance criteria from seeded fixtures.
- **ArchitectAgent** — AutonomousAgent that produces design decisions and component diagrams from fixtures.
- **CoderAgent** — AutonomousAgent that drafts or revises code artefacts as diff-shaped text.
- **ReviewerAgent** — AutonomousAgent that returns review findings against an allow-listed checklist.
- **PlanWorkflow** — Workflow with a plan → dispatch → execute → record → decide loop, replan branch, and terminal exit states.
- **PlanEntity** — EventSourcedEntity holding the plan lifecycle, both ledgers, and the final deliverable.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **PlanView** — projection used by the UI.
- **PlanEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the request prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Plan` record fields (e.g., add `complexityScore`).
- `prompts/planner.md` — narrow the planner to a specific SDLC phase (e.g., design-only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-task quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a request → planner decomposes, dispatches to specialists, completes within ~3 minutes.
2. Inject a tool-policy violation → guardrail blocks the offending task; planner records the block and replans.
3. Trigger the operator halt → no new tasks dispatch; in-flight tasks finish; plan moves to `HALTED`.
4. A specialist result containing a secret-shaped string is scrubbed before it reaches the planner's next-step prompt.

## License

Apache 2.0.
