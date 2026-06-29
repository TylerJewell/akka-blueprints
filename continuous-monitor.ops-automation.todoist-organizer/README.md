# Akka Sample: Todoist AI Inbox Organizer

A background worker polls the Todoist inbox at a fixed interval, classifies each task into a project and label set using an AI classifier, and writes the classification back via the Todoist API. Every mutation is held behind a before-tool-call guardrail, and a periodic eval sampler scores classification quality over time. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (before-tool-call guardrail, eval-periodic).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** (an in-process Todoist simulator provides canned tasks; swap `TodoistClient` for the real REST client when deploying).

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.todoist-organizer  ~/my-projects/todoist-organizer
cd ~/my-projects/todoist-organizer
```

(Optional) Edit `SPEC.md` to point `TodoistPoller` at the real Todoist REST API by swapping the simulated client, or to keep the in-process simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TodoistPoller** — TimedAction firing every 30 s that fetches uncategorized tasks from the Todoist inbox simulator.
- **TaskClassifierAgent** — Agent that assigns each task a target project, a priority level, and up to three labels.
- **TaskMutationGuardrail** — before-tool-call hook that blocks any `updateTask` call unless the classification meets safety thresholds.
- **TodoistTaskEntity** — EventSourcedEntity holding each task's lifecycle (fetched → classified → updated / skipped / failed).
- **OrganizerWorkflow** — per-task Workflow orchestrating classify → guard-check → update.
- **OrganizerView + OrganizerEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalRunner** — TimedAction running every 60 minutes; samples updated tasks and scores classification correctness.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the in-process simulator for the real Todoist API by replacing `TodoistSimulator` with a live `TodoistRestClient`.
- `SPEC.md §5` — add org-specific fields to the `TodoistTask` record (`teamId`, `duePolicy`, etc.).
- `prompts/task-classifier.md` — narrow the label taxonomy to your workspace's actual project list.
- `eval-matrix.yaml` — wire a real quality threshold under the guardrail's implementation block.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated task arrives → it is classified → guardrail passes → task is updated to the target project.
2. A task whose classification confidence is below threshold triggers the guardrail → the update is blocked → task transitions to SKIPPED.
3. Eval scoring runs and surfaces a classification score on updated tasks.
4. The mutation log shows no raw Todoist API calls that bypassed the guardrail.

## License

Apache 2.0.
