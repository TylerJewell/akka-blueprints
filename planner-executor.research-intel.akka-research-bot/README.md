# Akka Sample: Research Bot (Planner/Searcher/Writer)

A PlannerAgent breaks a research question into a search plan, a SearcherAgent executes each query in parallel, and a WriterAgent synthesizes the collected results into a cited report — all orchestrated as a durable workflow with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Web searches are answered from seeded fixtures bundled with the service.

## Generate the system

```sh
cp -r ./planner-executor.research-intel.akka-research-bot  ~/my-projects/akka-research-bot
cd ~/my-projects/akka-research-bot
```

(Optional) Edit `SPEC.md` to change the research topics, model provider, or the writer's output format.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that takes a research question and produces a `ResearchPlan` (list of targeted search queries, each tagged with an expected knowledge contribution).
- **SearcherAgent** — AutonomousAgent that executes a single search query against seeded fixtures and returns a `SearchResult`.
- **WriterAgent** — AutonomousAgent that synthesizes all collected search results into a `ResearchReport` with a narrative summary and a bibliography.
- **ResearchWorkflow** — Workflow with a plan → parallel-search → collect → write → publish loop, plus a replan branch and terminal exit states.
- **ResearchJobEntity** — EventSourcedEntity holding the job lifecycle, the research plan, the collected search results, and the final report.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **ResearchJobView** — projection used by the UI.
- **JobRequestConsumer** — Consumer that starts a workflow per submitted job.
- **JobQueue** — EventSourcedEntity that acts as an audit log of submissions.
- **JobSimulator** — TimedAction that drips a sample job every 90 s so the App UI is never empty.
- **StaleJobMonitor** — TimedAction that marks jobs stuck in `SEARCHING` for more than 5 minutes.
- **JobEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the research topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ResearchJob` record fields (e.g., add `confidenceScore`).
- `prompts/planner.md` — restrict the planner to a specific domain (e.g., regulatory research only).
- `eval-matrix.yaml` — add an `eval-event` control if you want per-result quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a research question → planner produces a search plan, searcher executes queries, writer synthesizes a report within ~3 minutes.
2. Submit a question whose plan would query a disallowed topic → guardrail blocks the offending query before the searcher runs; the planner revises the plan.
3. Submit a job and click **Halt new searches** while it is in `SEARCHING` → in-flight searches finish; no further queries dispatch; job moves to `HALTED`.
4. A search result containing a secret-shaped string is scrubbed before it reaches the writer's synthesis prompt.

## License

Apache 2.0.
