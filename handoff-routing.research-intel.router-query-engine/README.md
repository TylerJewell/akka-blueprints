# Akka Sample: Router Query Engine Workflow

A `RouterAgent` classifies an incoming research question, then hands the same `ANSWER` task off to either a `StructuredDataEngine` or a `SemanticSearchEngine` that owns the retrieval and synthesis end-to-end. Demonstrates the **handoff-routing** coordination pattern with an on-decision eval that scores every routing call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound question stream and the retrieval engines are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.research-intel.router-query-engine  ~/my-projects/router-query-engine
cd ~/my-projects/router-query-engine
```

(Optional) Edit `SPEC.md` to point `QuestionSimulator` at a real document corpus or to add a third engine (e.g. a graph-traversal engine for relationship queries).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QuestionSimulator** — TimedAction firing every 30 s that drips canned research questions from a JSONL file into `QuestionQueue`.
- **QuestionQueue** — EventSourcedEntity append-only log of every inbound question (audit before any LLM call).
- **RouterAgent** — typed Agent that classifies each question into `STRUCTURED`, `SEMANTIC`, or `UNCLEAR`.
- **StructuredDataEngine** — AutonomousAgent that owns the `ANSWER` task for structured-data questions (aggregations, filters, precise lookups).
- **SemanticSearchEngine** — AutonomousAgent that owns the `ANSWER` task for semantic questions (concept search, summarization, open-ended synthesis).
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every routing call against a 1–5 rubric.
- **QueryWorkflow** — Workflow per question: route → branch → retrieve-and-answer → publish.
- **QueryEntity** — EventSourcedEntity holding each question's lifecycle.
- **QueryView + QueryEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `RoutingDecided` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add a third engine (e.g. `GraphTraversalEngine` for entity-relationship queries) and extend the routing taxonomy.
- `SPEC.md §5` — extend the `Query` record with deployer-specific fields (`corpusId`, `userTier`, `queryLanguage`).
- `prompts/router-agent.md` — adjust routing thresholds or add per-corpus routing rules.
- `prompts/structured-data-engine.md` / `prompts/semantic-search-engine.md` — bind the engine prompt to a real corpus schema or embedding model.
- `eval-matrix.yaml` — add a regulation anchor if your deployment falls under a sector-specific data-governance requirement.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a structured-data question → routed to `STRUCTURED` → answered by `StructuredDataEngine` → published.
2. Simulator drips a semantic question → routed to `SEMANTIC` → answered by `SemanticSearchEngine` → published.
3. An ambiguous question routes to `UNCLEAR` and the workflow terminates in `ESCALATED` without invoking either engine.
4. The routing score (1–5) and rationale appear on every routed question within a few seconds of the routing decision.
5. Manually submitted questions via `POST /api/queries` follow the same path as simulated questions.

## License

Apache 2.0.
