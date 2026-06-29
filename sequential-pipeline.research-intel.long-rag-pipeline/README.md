# Akka Sample: Long RAG Workflow

A single `DocumentAgent` walks a research query through three task phases — **RETRIEVE → SYNTHESIZE → REPORT** — wired together by explicit task dependencies tuned for very long documents. Each phase receives a typed input, produces a typed output, and operates on a scoped chunk window. The user submits a query and receives a structured `ResearchReport`.

Demonstrates the **sequential-pipeline** coordination pattern with governance around long-context generation: a `before-llm-response` guardrail that checks every generated passage for grounded citation coverage before it leaves the agent, and an `on-decision-eval` evaluator that scores every emitted report for factual traceability and chunk coverage.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every retrieve / synthesize / report tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.research-intel.long-rag-pipeline  ~/my-projects/long-rag-workflow
cd ~/my-projects/long-rag-workflow
```

(Optional) Edit `SPEC.md` to point at a different document corpus seed list, a different model provider, or a richer set of synthesis tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DocumentAgent** — one AutonomousAgent declaring three Task constants (`RETRIEVE_CHUNKS`, `SYNTHESIZE_FINDINGS`, `WRITE_REPORT`); the workflow runs them in order, passing each typed output forward as the next task's instruction context.
- **LongRagPipelineWorkflow** — runs `retrieveStep → synthesizeStep → reportStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `QueryEntity` before the next step starts.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (`ChunksRetrieved`, `FindingsSynthesized`, `ReportWritten`, `EvaluationScored`).
- **RetrieveTools / SynthesizeTools / ReportTools** — three function-tool classes registered on the agent, one per phase. Extended chunking strategies (sliding-window overlap, long-passage embedding buckets) are implemented in `RetrieveTools`.
- **CitationGuardrail** — the `after-llm-response` guardrail that checks every generated passage for at least one grounded citation before the response leaves the agent. Passages that cite no chunk in the active `ChunkWindow` are flagged and the agent is instructed to revise.
- **CoverageScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `ReportWritten` and emits a 1–5 score based on chunk coverage, citation count, and section coherence.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded query set under `src/main/resources/sample-events/queries.jsonl` to fit your target research domain.
- `SPEC.md §4` and `prompts/document-agent.md` — narrow the agent's role (e.g., constrain it to legal-document review, to scientific literature synthesis, to technical specification analysis) by tightening the system prompt and renaming the typed records (`Finding`, `Theme`, `ReportSection`).
- `SPEC.md §5` — extend the typed outputs (`ChunkWindow`, `Synthesis`, `ResearchReport`) with domain-specific fields. The citation guardrail does not need editing — it checks passage-level citation presence, not field shapes.
- `eval-matrix.yaml` — wire a real embedding-similarity evaluator (replace the deterministic stub with a vector-overlap check against the active `ChunkWindow`) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a research query → `RETRIEVE` runs → `SYNTHESIZE` runs → `REPORT` runs → a typed `ResearchReport` lands in the UI within ~90 s. Every transition is visible in real time.
2. A generated passage cites no chunk from the active `ChunkWindow` (forced via the mock LLM) → the `after-llm-response` guardrail flags the passage → the agent revises with a citation → the pipeline completes correctly.
3. Every `ResearchReport` emitted has an on-decision coverage score visible on the same UI card; reports whose sections cite fewer than two distinct chunks receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the RETRIEVE task does not see report instructions, and the REPORT task does not see raw chunk embeddings — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
