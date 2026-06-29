# Architecture — multiformat-hybrid-rag

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question, writes a `QuerySubmitted` event onto `QueryEntity`, and starts a `QueryWorkflow` instance. The workflow's `retrieveStep` calls `ChunkStore.topK` — a pure in-process keyword-overlap search — to retrieve the most relevant chunks. No embedding service, no external index. The `ChunkStore` was populated at startup by `ChunkIndexer`, which ingested the seeded `corpus-chunks.jsonl` and normalised each entry's content by format.

The workflow's `answerStep` calls `ResearchAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and each retrieved chunk as a separate `TaskDef.attachment(...)` file. The agent's `before-agent-response` guardrail (`CitationGuardrail`) validates each candidate answer before it leaves the agent loop. Once an answer passes, the workflow writes `AnswerRecorded` to the entity. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `ChunkStore` is a deterministic retriever. `CitationGuardrail` is a deterministic checker. Only `ResearchAgent` talks to a model. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Two synchronous segments and one asynchronous one:

1. `retrieveStep` is synchronous and in-process — `ChunkStore.topK` runs in milliseconds, so `RetrievalCompleted` lands almost immediately after submission.
2. `answerStep` is bounded by a 60 s timeout to accommodate model latency. The step starts after `RetrievalCompleted` lands on the entity.
3. SSE projection from `QueryEntity` to the browser is continuous — every state transition appears as a separate event on the `/api/queries/sse` stream.

## State machine

Six states. The interesting paths:

- The happy path with an answer is `SUBMITTED → RETRIEVING → ANSWERING → ANSWERED`.
- When no chunk is relevant, the agent returns `decision = NO_RESULT` and the path is `SUBMITTED → RETRIEVING → ANSWERING → NO_RESULT`.
- Two failure transitions land in `FAILED`: a retrieval error during `SUBMITTED`, and an agent error or guardrail-exhaustion during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity; the UI shows the partial state so the researcher can diagnose the failure.
- There is no `ACCEPTED` or `REJECTED` state. The answer is informational; the researcher reads it and acts outside the system. The blueprint deliberately stops at `ANSWERED` or `NO_RESULT`.

## Entity model

`QueryEntity` is the source of truth. It emits five event types. `QueryView` projects every event into a row used by the UI. `ChunkIndexer` writes to `ChunkStore` independently of the query lifecycle — it is a startup-time indexing step, not a per-query step. `QueryWorkflow` both reads (`getQuery`) and writes (`attachRetrieval`, `markAnswering`, `recordAnswer`, `fail`) on the entity. The relationship between `ResearchAgent` and `ResearchAnswer` is "returns" — the agent's task result is the answer record.

## Governance flow

For any answer that lands in the entity log, it passed:

1. **ChunkStore retrieval** — keyword-overlap ranking ensures only corpus-sourced passages reach the agent. The agent cannot cite sources it was not given.
2. **ResearchAgent** — one model call, one structured output with citations.
3. **before-agent-response guardrail** — citation chunkIds are cross-checked against the retrieved set; uncited answerText and citation-free NO_RESULT are caught before the response leaves the agent loop.

Each step is independent. The guardrail cannot compensate for a bad retrieval; the retrieval cannot compensate for a guardrail bypass. Removing either step opens an explicit gap the other does not silently cover.
