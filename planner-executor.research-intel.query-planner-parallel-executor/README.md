# Akka Sample: Query Planning Workflow

A Planner decomposes a complex research query into parallel sub-queries, dispatches them to concurrent retrieval executors, streams partial results as they arrive, and conditionally loops to issue follow-up sub-queries when the initial coverage is insufficient. Demonstrates the **planner-executor** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** Retrieval endpoints — corpus search, web fixture lookup, knowledge-base keyword index — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.research-intel.query-planner-parallel-executor  ~/my-projects/query-planner-parallel-executor
cd ~/my-projects/query-planner-parallel-executor
```

(Optional) Edit `SPEC.md` to change the query corpus, adjust the fan-out concurrency limit, or narrow the planner to a specific research domain.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that reads an incoming research question, drafts a `QueryPlan` (decomposed sub-queries with retrieval strategy tags), and revises the plan when follow-up rounds are required.
- **CorpusSearchExecutor** — AutonomousAgent that runs sub-queries against the seeded document corpus fixtures.
- **WebLookupExecutor** — AutonomousAgent that answers sub-queries from seeded web-fixture data.
- **KnowledgeBaseExecutor** — AutonomousAgent that queries the seeded keyword-index fixtures.
- **SynthesisAgent** — AutonomousAgent that merges partial results across completed sub-queries into a `ResearchAnswer`.
- **QueryWorkflow** — Workflow with a plan → fan-out → collect → evaluate → decide loop, follow-up branch, and terminal exit states.
- **QuerySessionEntity** — EventSourcedEntity holding the session lifecycle, query plan, sub-query results, and final answer.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **QueryQueue** — EventSourcedEntity audit log of submitted queries.
- **SessionView** — projection used by the UI.
- **QueryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `QuerySession` record (e.g., add a `confidenceThreshold` field).
- `prompts/planner.md` — restrict the planner to a specific domain (e.g., legal research only).
- `eval-matrix.yaml` — tighten the guardrail allow-list or adjust the plan-quality eval threshold.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a multi-faceted research question → planner decomposes it, executors run in parallel, synthesis produces a cited answer within ~3 minutes.
2. A sub-query plan that would exceed the retrieval allow-list is blocked by the guardrail; the planner revises.
3. The plan-quality eval fires before fan-out; a low-quality plan is rejected and the planner retries.
4. Submit a query while a session is executing → operator halts new dispatches; in-flight sub-queries finish; session moves to `HALTED`.
5. A retrieval result containing a credential-shaped string is scrubbed before reaching the synthesis prompt.

## License

Apache 2.0.
