# Akka Sample: Basic RAG Workflow

A single `RagAgent` walks a user query through two task phases — **INGEST → QUERY** — wired together by explicit task dependencies. The INGEST phase indexes a document corpus into an in-process vector store; the QUERY phase retrieves relevant chunks and composes a grounded answer. The user submits a question and receives a structured `RagAnswer`.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: a `before-agent-response` guardrail that filters the agent's answer before it is returned to the caller, blocking responses that contain no grounded citations or that reference content not present in the indexed corpus.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the document corpus, vector store, and retrieval logic are all implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.basic-rag-pipeline  ~/my-projects/basic-rag
cd ~/my-projects/basic-rag
```

(Optional) Edit `SPEC.md` to point at a different document corpus, a different model provider, or a richer retrieval strategy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RagAgent** — one AutonomousAgent declaring two Task constants (`INGEST_CORPUS`, `ANSWER_QUERY`); the workflow runs them in order, feeding the indexed corpus state forward as the QUERY task's context.
- **RagPipelineWorkflow** — runs `ingestStep → queryStep → guardrailStep`. Each step calls `runSingleTask` and writes the typed result back onto `RagSessionEntity` before the next step starts.
- **RagSessionEntity** — an EventSourcedEntity holding the per-session lifecycle (`CorpusIndexed`, `QueryAnswered`, `GuardrailApplied`, `AnswerBlocked`).
- **IngestTools / QueryTools** — two function-tool classes registered on the agent, one per phase. The in-process `VectorStore` backs both.
- **AnswerGuardrail** — the `before-agent-response` hook that filters the final answer before it is returned to the caller. Any answer whose citations do not trace back to indexed chunks is blocked; a structured rejection is recorded on the entity.
- **RagSessionView + RagEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded document set under `src/main/resources/sample-data/corpus/*.json` to fit your demo domain.
- `SPEC.md §4` and `prompts/rag-agent.md` — narrow the agent's retrieval strategy (e.g., restrict it to a single document type, add a re-ranker step) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`IndexedCorpus`, `RagAnswer`) with domain-specific fields.
- `eval-matrix.yaml` — the `G1` guardrail is pre-wired as blocking; switch `enforcement` to `non-blocking` if you want to observe unfiltered answers in development.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → `INGEST` runs → `QUERY` runs → a typed `RagAnswer` with at least one citation lands in the UI within ~45 s. Every transition is visible in real time.
2. The mock LLM produces an answer that cites a URL not present in the indexed corpus → the `before-agent-response` guardrail blocks the answer → a `GuardrailApplied` event is recorded → the agent retries → the pipeline completes with a grounded answer.
3. A question with no relevant chunks in the corpus returns an honest "no relevant content found" answer rather than a hallucinated one.
4. Each task receives only its own typed inputs; the INGEST task does not see the query text, and the QUERY task does not see raw document bytes — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
