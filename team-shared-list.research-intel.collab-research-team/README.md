# Akka Sample: Collaborative Research Assistant

A user submits a research question; a coordinator agent fans it out to specialized researchers, each of which gathers sources and produces a findings report; a synthesis agent merges the findings into a single cited report, and an eval agent verifies every conclusion is grounded by at least one cited source before the report is finalized. Demonstrates the **team-shared-list** coordination pattern applied to multi-agent research with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The web search, source fetching, and synthesis are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.research-intel.collab-research-team  ~/my-projects/collab-research-team
cd ~/my-projects/collab-research-team
```

(Optional) Edit `SPEC.md` to change the researcher roster, the model provider, or the sample queries the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchCoordinator** — AutonomousAgent that breaks a research question into sub-topics and writes one `ResearchTask` per sub-topic onto the shared board.
- **ResearcherAgent** — AutonomousAgent (run as `researcher-1`, `researcher-2`, `researcher-3`) that claims a sub-topic, gathers sources, and returns a `FindingsReport`.
- **SynthesisAgent** — AutonomousAgent that merges all `FindingsReport` items for a question into a single `ResearchReport` with citations.
- **PlanningWorkflow** — Workflow that runs the coordinator and writes tasks onto the board.
- **ResearcherWorkflow** — one durable loop per researcher: poll the board, claim an open task atomically, gather findings, and mark the task done or blocked.
- **SynthesisWorkflow** — starts when all tasks for a question are done; runs the synthesis agent and records the final report.
- **ResearchTaskEntity** — one EventSourcedEntity per sub-topic task; atomic claim prevents two researchers grabbing the same task.
- **QuestionEntity**, **ResearcherMailbox**, **IntakeQueue** — question lifecycle, per-researcher coordination messages, and the submission log.
- **OperatorControl** — KeyValueEntity holding the operator halt flag.
- **TaskBoardView** — the shared task list researchers poll and the UI streams.
- **ResearchEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the researcher roster (`researcher-1`, `researcher-2`, `researcher-3`) to a different count.
- `prompts/researcher.md` — narrow the researcher to a specific domain (e.g., life sciences only).
- `eval-matrix.yaml` — extend the after-llm-response guardrail's citation-density threshold if your domain requires stricter grounding.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → the coordinator writes sub-topic tasks → researchers claim and complete them → synthesis produces a final report.
2. Two researchers poll at the same instant → each sub-topic task is claimed by exactly one researcher; no double-claim.
3. A researcher raises a coordination request → the task goes `BLOCKED`, a message lands in the peer's mailbox, the reply unblocks it.
4. The synthesis agent produces a report whose conclusion lacks a citation → the after-llm-response guardrail blocks the report and the question is flagged for review.
5. The operator halts the team → claiming pauses and tool calls are refused; resume continues the research.

## License

Apache 2.0.
