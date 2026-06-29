# Architecture — chroma-rag-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question submission, writes a `QuerySubmitted` event onto `QueryEntity`, and immediately starts a `QueryWorkflow` instance. The workflow's `retrieveStep` calls `ChromaDbClient` — an in-process wrapper around the embedded ChromaDB Java client — to find the top-5 passages by cosine similarity to the question. The retrieved chunks are recorded on the entity via `recordChunks`, then passed to `RagAgent` — the single AutonomousAgent — as a JSON block inside the `TaskDef.instructions(...)` field. The agent's `before-agent-response` guardrail (`CitationGuardrail`) validates each candidate response against the retrieved chunk list. Once an answer passes, the workflow writes `AnswerRecorded` and runs `GroundednessScorer` in `evalStep`. The score lands as `GroundednessScored`. `RagView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`GroundednessScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

The `CorpusIndexer` Consumer runs once at startup to embed and upsert the bundled docs corpus into ChromaDB. After that, it remains idle unless a new corpus-load event arrives.

## Interaction sequence

The sequence traces the happy path (J1). Key timing points:

1. The `retrieveStep` call to ChromaDB is in-process — sub-millisecond in normal operation. The 20 s timeout on `retrieveStep` covers pathological index-cold-start scenarios.
2. The `answerStep` call to `RagAgent` is bounded by the 60 s step timeout. The agent's `maxIterationsPerTask(3)` caps guardrail-triggered retries without unbounded runtime.
3. The `evalStep` is synchronous and deterministic — `GroundednessScorer` finishes in milliseconds with no external call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → RETRIEVING → ANSWERING → ANSWERED → EVALUATED`.
- Two failure transitions land in `FAILED`: a retrieval error during `RETRIEVING`, and an agent error (or guardrail-exhaustion) during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity — any retrieved chunks are available for inspection.
- There is no `VERIFIED` or `APPROVED` state. The answer is informational; the user reads it and acts outside the system. The blueprint stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `RagView` projects every event into a row for the UI. `CorpusIndexer` subscribes to corpus-init events. `QueryWorkflow` both reads (`getQuery`) and writes (`recordChunks`, `markAnswering`, `recordAnswer`, `recordGroundedness`, `fail`) on the entity. The relationship between `RagAgent` and `RagAnswer` is "returns" — the agent's task result is the answer record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the question passed through:

1. **ChromaDB retrieval** — the agent sees only the top-k relevant passages, not the full corpus.
2. **RagAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — hallucinated chunk IDs and empty-citation answers are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every grounded answer still receives a 1–5 groundedness score so the user knows which answers are well-evidenced.

Each step is independent. Removing the guardrail exposes the citation hallucination risk; removing the eval removes the signal for under-evidenced answers. The retrieval step cannot substitute for either.
