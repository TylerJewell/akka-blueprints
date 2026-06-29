# Architecture — corrective-rag-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`RagEndpoint` is the entry point. It writes a `QueryCreated` event directly to `QueryEntity` (event-sourced for lifecycle tracking). A `Consumer` subscribes to that event and starts a `CorrectiveRagWorkflow` per submission. The workflow sequences four agents in order: `RetrieverAgent` fetches chunks from `CorpusStore`, `RelevanceEvaluatorAgent` judges whether the context is sufficient, and — only on `INSUFFICIENT` — a deterministic guardrail step runs before `WebSearchAgent` fires the external fallback. `AnswerSynthesizerAgent` generates the final answer from whatever context is assembled. Each step emits events on `QueryEntity`. `QueryView` projects those events into the read model the UI streams via SSE.

Two TimedActions operate alongside: `QuerySimulator` drips a sample query every 60 seconds so the UI is not empty on first load; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any retrieval cycle whose relevance verdict has not yet been sampled. `CorpusStore` is an in-process entity pre-seeded from a JSONL file at bootstrap.

## Interaction sequence

The sequence diagram traces a J2-style corrective-fallback path: the first retrieval returns low-scoring chunks, the evaluator returns `INSUFFICIENT`, the guardrail clears, web search fires, and the synthesizer produces a `RETRIEVAL_PLUS_WEB` answer. Each step's events are written to `QueryEntity` before the next step begins, so the UI's per-attempt timeline reconstructs the full pipeline.

## State machine

The query moves through two transient states (`RETRIEVING`, `EVALUATING`, `WEB_SEARCHING`) and two terminal states (`ANSWERED`, `ANSWER_DEGRADED`). The `EVALUATING → RETRIEVING` back-edge represents the re-retrieval path when the corpus is insufficient but the attempt ceiling has not been reached. The `EVALUATING → WEB_SEARCHING` edge fires when the ceiling is reached and the guardrail clears. Both terminal states preserve all attempts, all verdicts, and the final answer or degradation reason on the entity.

## Entity model

`QueryEntity` is the system's source of truth; every pipeline transition writes one of nine event types. `CorpusStore` is a supporting entity that holds the in-process document corpus and is never directly queried by the UI. `QueryView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `retrieveStep`, `evaluateStep`, `webSearchStep`, and `synthesizeStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide bound: the loop terminates at `maxRetrievalAttempts` (default 3) before re-querying the corpus. A `SUFFICIENT` verdict short-circuits the loop at any attempt.
- Default step recovery: `maxRetries(2).failoverTo(degradeStep)` — any unrecoverable agent failure ends in `ANSWER_DEGRADED`, not a hung workflow.
- Idempotency: `RagEndpoint.submit` deduplicates on `(question, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(queryId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- CorpusStore: pre-seeded at bootstrap from `corpus-chunks.jsonl`; the `AddChunk` command is idempotent on `chunkId`, so restarts do not duplicate entries.
