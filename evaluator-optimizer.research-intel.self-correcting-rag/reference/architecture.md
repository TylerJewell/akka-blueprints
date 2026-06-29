# Architecture — self-correcting-rag

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`RagEndpoint` is the entry point. A submitted query is written directly to `QueryEntity` (status `RETRIEVING`). A `Consumer` subscribes to `QueryCreated` events and starts a `RagWorkflow` per query. The workflow calls five agents in sequence: `RetrieverAgent` fetches candidates from `CorpusEntity`, `RelevanceGraderAgent` filters them, `QueryRewriterAgent` rewrites the query when grading yields no relevant documents, `GeneratorAgent` produces a grounded answer from the retained documents, and `HallucinationGraderAgent` verifies the answer against those documents before it is surfaced. Each step emits an event on `QueryEntity`. `QueryView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `QuerySimulator` drips a sample query every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any grading step (relevance or hallucination) that completed and has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style single-pass answer: retrieval returns relevant documents, grading retains them, the generator answers, and the hallucination grader confirms the answer is grounded. No query rewrite occurs. Each step's events are written to `QueryEntity` before the next step begins, so the UI's per-pass timeline reconstructs the full pipeline exactly.

## State machine

The query moves through four transient states (`RETRIEVING`, `GRADING`, `GENERATING`, `HALLUCINATION_CHECK`) and three terminal states (`ANSWERED`, `FAILED_NO_RELEVANT_DOCS`, `FAILED_HALLUCINATION`). The `GRADING → REWRITING → RETRIEVING` cycle is the adaptive correction path: triggered when the grader finds no relevant documents, it rewrites the query and retries retrieval up to `maxRewriteAttempts` before the `FAILED_NO_RELEVANT_DOCS` terminal. The `HALLUCINATION_CHECK → GENERATING` back-edge is the hallucination correction path: triggered when the grader returns `HALLUCINATED`, it re-generates up to `maxGenerationAttempts` before the `FAILED_HALLUCINATION` terminal.

## Entity model

`QueryEntity` is the system's source of truth; every pipeline transition writes one of eleven event types. `CorpusEntity` stores the document corpus that the workflow reads via `RetrieverAgent`. `QueryView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for all five agent-calling steps; the workflow's internal routing logic is in-process and effectively instant.
- Rewrite ceiling: `maxRewriteAttempts` (default 2). The workflow checks the count before scheduling a rewrite; it never recurses past the ceiling.
- Generation ceiling: `maxGenerationAttempts` (default 2). Same guard pattern applied before routing back to `generateStep`.
- Default step recovery: `maxRetries(2).failoverTo(failHallucinationStep)` — any unrecoverable agent failure ends in `FAILED_HALLUCINATION`, not in a hung workflow.
- EvalSampler idempotency: keyed on `(queryId, passNumber, stepKind)` so a double-fire tick is a no-op on the entity side.
- CorpusEntity bootstrapping: the corpus is loaded at startup from `src/main/resources/sample-events/corpus-documents.jsonl`; no external vector store is required.
