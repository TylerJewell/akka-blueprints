# Architecture — pgvector-rag-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question submission, writes a `QuestionSubmitted` event onto `QuestionEntity`, and starts a `QueryWorkflow` instance. The workflow's `embedStep` converts the question text into a float vector using `EmbeddingService`. The `retrieveStep` queries pgvector's `corpus_chunks` table for the top-5 cosine-nearest chunks using `PgVectorStore`. The `answerStep` calls `CorpusQueryAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and the retrieved passages as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`CitationGuardrail`) validates each candidate response. Once an answer passes, the workflow writes `AnswerRecorded` and runs `AnswerEvaluationScorer` in `evalStep`. The score lands as `EvaluationScored`. `QuestionView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

Alongside the question path, `CorpusEntity` tracks each ingested document. `CorpusIngestionConsumer` subscribes to `DocumentAdded` events, splits the document into overlapping chunks using `ChunkSplitter`, embeds each chunk via `EmbeddingService`, and stores the vectors in pgvector. The corpus path and the question path share the same `EmbeddingService` and `PgVectorStore` but operate on independent entity lifecycles.

The graph has exactly one AutonomousAgent. `AnswerEvaluationScorer` is a deterministic rule-based scorer — it makes no LLM call. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Note the distinct stages where the system waits on external I/O:

1. The `embedStep` waits on the embedding endpoint (model provider or mock). Sub-second for the mock; 1–3 s for a live endpoint.
2. The `retrieveStep` queries pgvector. Sub-second with the seeded corpus; index size determines latency at scale.
3. The `answerStep`'s `runSingleTask` waits on the LLM completion. Bounded by the 60 s step timeout.

The `evalStep` is synchronous and in-process — it finishes in milliseconds.

## State machine

Six states for `QuestionEntity`. The interesting paths:

- The happy path is `SUBMITTED → RETRIEVING → ANSWERING → ANSWERED → EVALUATED`.
- Three failure transitions land in `FAILED`: an embed error during `SUBMITTED`, a retrieval error during `RETRIEVING`, and an agent error (or guardrail-exhaustion) during `ANSWERING`. A `FAILED` question's prior data is preserved on the entity.
- There is no `ACCEPTED` or `REJECTED` state. The answer is informational; the researcher reads it and acts outside the system. The blueprint stops at `EVALUATED`.

## Entity model

`QuestionEntity` is the per-question source of truth; `CorpusEntity` is the per-document corpus source of truth. Both emit event streams. `QuestionView` projects `QuestionEntity` events into the read model. `CorpusIngestionConsumer` subscribes to `CorpusEntity` events to drive the indexing pipeline. `QueryWorkflow` both reads (`getQuestion`) and writes (`markRetrieving`, `markAnswering`, `recordAnswer`, `recordEvaluation`, `fail`) on `QuestionEntity`. `CorpusQueryAgent`'s task result is the `Answer` record.

## Grounding enforcement flow

For any answer that lands in the entity log, the question passed through:

1. **`embedStep`** — the question is converted to a vector; retrieval is semantically anchored to the corpus, not to the model's prior.
2. **`retrieveStep`** — the five most relevant chunks are fetched from pgvector and attached to the task; the agent cannot access other corpus content.
3. **`CorpusQueryAgent`** — one model call, one structured output, with passages passed as an attachment.
4. **`before-agent-response` guardrail** — uncited answers and fabricated chunk id references are caught before the response leaves the agent loop.
5. **On-decision evaluator** — every well-formed answer still gets a 1–5 citation-density score so researchers know which answers are thinly supported.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
