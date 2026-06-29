# Akka Sample: Citation Query Engine

A `CitationAgent` walks a research query through three task phases — **RETRIEVE → ATTRIBUTE → COMPOSE** — producing a grounded answer where every factual claim is pinned to a cited source passage. Each phase runs as its own typed task with its own scoped tool set; no phase sees the raw artifacts of another until they are handed forward as a typed result.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: an `after-llm-response` guardrail that verifies every claim in the composed answer carries a citation before the response is released, and an `on-decision-eval` evaluator that scores citation coverage across the full answer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every retrieve / attribute / compose tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.research-intel.citation-rag  ~/my-projects/citation-rag
cd ~/my-projects/citation-rag
```

(Optional) Edit `SPEC.md` to point at a different document corpus, a different model provider, or a richer set of retrieval tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CitationAgent** — one AutonomousAgent declaring three Task constants (`RETRIEVE_PASSAGES`, `ATTRIBUTE_CLAIMS`, `COMPOSE_ANSWER`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **CitationQueryWorkflow** — runs `retrieveStep → attributeStep → composeStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `QueryEntity` before the next step starts.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (`PassagesRetrieved`, `ClaimsAttributed`, `AnswerComposed`, `CitationScored`).
- **RetrieveTools / AttributeTools / ComposeTools** — three function-tool classes registered on the agent, one per phase. The `after-llm-response` guardrail enforces that every claim in the composed answer carries a citation before release.
- **CitationGuardrail** — the runtime check that verifies the COMPOSE phase output before it is committed. A composed answer with uncited claims is rejected and the agent retries within its iteration budget.
- **CoverageScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `AnswerComposed` and emits a 1–5 citation-coverage score.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded query set under `src/main/resources/sample-events/queries.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/citation-agent.md` — narrow the agent's role (e.g., constrain it to legal documents, to academic papers, to internal policy documents) by tightening the system prompt and renaming the typed records (`PassageSet`, `ClaimSet`, `Answer`).
- `SPEC.md §5` — extend the typed outputs (`PassageSet`, `ClaimSet`, `Answer`) with domain-specific fields. The after-llm-response guardrail does not need editing — it checks claim-to-citation linkage, not field shapes.
- `eval-matrix.yaml` — wire a real coverage evaluator (replace the deterministic stub with a semantic similarity check against a real corpus) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a query → `RETRIEVE` runs → `ATTRIBUTE` runs → `COMPOSE` runs → a typed `Answer` lands in the UI within ~60 s. Every transition is visible in real time.
2. The agent produces a COMPOSE-phase answer with a claim that has no corresponding citation (forced via the mock LLM) → the `after-llm-response` guardrail rejects it → the workflow records the rejection event → the agent retries and produces a fully-cited answer → the pipeline completes correctly.
3. Every `Answer` emitted has an on-decision citation-coverage score visible on the same UI card; answers with uncited claims receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the RETRIEVE task does not see compose instructions, and the COMPOSE task does not see raw chunk fetches — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
