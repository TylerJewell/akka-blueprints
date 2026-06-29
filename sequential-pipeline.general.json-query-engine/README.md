# Akka Sample: JSON Query Engine Workflow

A single `QueryAgent` walks a user question through three task phases — **PARSE → TRAVERSE → RESPOND** — using LLM-generated path expressions to navigate semi-structured JSON data. Each phase has its own typed input, typed output, and a scoped set of path-resolution tools. The user submits a natural-language question and a JSON document identifier, and receives a structured `QueryResult`.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: a `before-tool-call` guardrail that validates every generated path expression before it is applied to the document store, blocking malformed or out-of-scope paths at the agent–tool boundary.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the JSON document store is an in-process corpus loaded from `src/main/resources/sample-data/documents/`.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.json-query-engine  ~/my-projects/json-query-engine
cd ~/my-projects/json-query-engine
```

(Optional) Edit `SPEC.md` to point at a different JSON document set, a different model provider, or a richer set of path tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QueryAgent** — one AutonomousAgent declaring three Task constants (`PARSE_QUESTION`, `TRAVERSE_DOCUMENT`, `COMPOSE_RESPONSE`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **JsonQueryWorkflow** — runs `parseStep → traverseStep → respondStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `QueryEntity` before the next step starts.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (`QuestionParsed`, `PathsTraversed`, `ResponseComposed`, `AccuracyScored`).
- **ParseTools / TraverseTools / RespondTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that generated path expressions pass structural validation before execution.
- **PathGuardrail** — the runtime check that backs the path-expression contract. A path call whose expression does not pass structural validation (malformed syntax, references a non-existent root key, or exceeds the declared traversal depth) is rejected before the traversal runs.
- **AccuracyScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `ResponseComposed` and emits a 1–5 score measuring path coverage, answer grounding, and citation completeness.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded JSON documents under `src/main/resources/sample-data/documents/` to fit your demo data set (product catalogues, configuration trees, API response archives, etc.).
- `SPEC.md §4` and `prompts/query-agent.md` — narrow the agent's role (e.g., constrain it to a single document schema, or a restricted set of root keys) by editing the system prompt and the `DocumentSchema` record.
- `SPEC.md §5` — extend the typed outputs (`ParsedQuestion`, `TraversalResult`, `QueryResult`) with schema-specific fields. The path-validation guardrail does not need editing — it checks structural validity, not field shapes.
- `eval-matrix.yaml` — wire a real answer-grounding evaluator (replace the deterministic stub with a semantic-similarity check against the source JSON values) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question and a document id → `PARSE` runs → `TRAVERSE` runs → `RESPOND` runs → a typed `QueryResult` lands in the UI within ~60 s. Every transition is visible in real time.
2. The agent generates a malformed path expression (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries with a corrected path → the pipeline completes correctly.
3. Every `QueryResult` emitted has an on-decision accuracy score visible on the same UI card; results that cite a JSON value not found in the traversal paths receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the PARSE task does not see the raw document, and the TRAVERSE task does not see the compose instructions — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
