# Akka Sample: Project Manager Agent (Planner)

A continuous background worker polls a project board, assigns newly created tasks to the right owners, chases overdue items with automated nudges, and holds every outbound assignment or escalation for a before-tool-call guardrail check before the action is dispatched. Demonstrates the **continuous-monitor** coordination pattern with a before-tool-call guardrail protecting task-assignment and nudge actions that affect humans.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — the system ships with an in-memory Planner simulator. No live Microsoft 365 account required to run out of the box.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.project-tracker  ~/my-projects/project-tracker
cd ~/my-projects/project-tracker
```

(Optional) Edit `SPEC.md §3` to point `PlannerPoller` at a real Microsoft Graph API endpoint, or keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerPoller** — TimedAction firing every 20 s that reads new and updated tasks from a simulated Planner board.
- **BoardSyncConsumer** — Consumer that normalises raw board events and emits `TaskCreated` / `TaskUpdated` on `PlannerTaskEntity`.
- **AssignmentAgent** — Agent (typed) that recommends a team member for each unassigned task.
- **NudgeAgent** — Agent (typed) that drafts a plain-text nudge message for overdue tasks.
- **PlannerTaskEntity** — EventSourcedEntity holding each task's lifecycle (created → assigned → in-progress → completed / overdue / escalated).
- **StaleTaskChecker** — TimedAction firing every 5 minutes that finds overdue tasks and triggers the nudge flow.
- **PlannerWorkflow** — per-task Workflow orchestrating classify → assign → dispatch.
- **BoardView + BoardEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 60 minutes; samples completed tasks, scores assignment quality.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated board for a real Microsoft Graph calls.
- `SPEC.md §5` — extend the `PlannerTask` record with org-specific fields (`costCentre`, `sprintId`, `customerImpact`).
- `prompts/assignment-agent.md` — narrow the assignment logic to your team's skill taxonomy.
- `prompts/nudge-agent.md` — adjust tone or add escalation thresholds.
- `eval-matrix.yaml` — add `regulation_anchors` once you have identified your applicable frameworks.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated task appears on the board → gets assigned → moves to IN_PROGRESS.
2. A task passes its due date → a nudge is drafted → held until the guardrail check passes.
3. A nudge is dispatched (simulated outbound); the entity records the dispatch timestamp.
4. Eval scoring runs within 60 minutes and surfaces an assignment-quality score.

## License

Apache 2.0.
