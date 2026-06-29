# Architecture — rag-citation-answerer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph shows two parallel pipelines that converge at the agent. The **ingest pipeline** flows from `QueryEndpoint` → `DocumentEntity` → `DocumentSanitizer` → `ChunkIndexer` → `ChunkStore`. The **query pipeline** flows from `QueryEndpoint` → `QueryEntity` → `AnswerWorkflow` → `ChunkStore` (retrieval) → `CitationAnswererAgent` → back to `QueryEntity`. The two pipelines share no runtime lock; a query may begin while documents are still being indexed. `QueryView` and `DocumentView` project their respective entity events into read-model rows; `QueryEndpoint` serves both read models and their SSE streams.

The graph deliberately has no second LLM call. `ChunkStore.retrieve` is a keyword-TF scorer — deterministic, in-process, no model invoked. `CitationGuardrail` is a structural validator — it checks chunkIds and excerpt presence without calling a model. This is what makes the blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1) for a document upload followed by a question. Note the two distinct pipeline phases:

1. The ingest phase: `DocumentUploaded` triggers `DocumentSanitizer` (sub-second), which triggers `ChunkIndexer` (typically under 2 s for a 1200-word document). The user can submit a query as soon as the document reaches `INDEXED`.
2. The query phase: `AnswerWorkflow` starts immediately on query submission. `retrieveStep` calls `ChunkStore.retrieve` synchronously — no network hop, completes in milliseconds. `answerStep` calls `CitationAnswererAgent`, which may take 10–30 s depending on the model provider.

The `before-agent-response` guardrail fires on every agent iteration. On a well-formed answer it is transparent; on a citation referencing an unknown chunkId it forces a retry inside the same task.

## State machines

Two entities have independent lifecycles.

**DocumentEntity** has five states: `UPLOADED → SANITIZED → CHUNKED → INDEXED` (happy path). `FAILED` is reachable from `UPLOADED` (sanitizer error) or `SANITIZED` (indexer error). There is no re-index command; if a document fails, the deployer re-uploads.

**QueryEntity** has four states: `SUBMITTED → RETRIEVING → ANSWERED` (happy path). `FAILED` is reachable from either `SUBMITTED` (retrieval error) or `RETRIEVING` (agent or guardrail-exhaustion error). A failed query preserves the `RetrievalStarted` event if retrieval succeeded before the agent error — the partial state is visible in the UI.

Both entities reach a terminal state (`INDEXED` / `ANSWERED` / `FAILED`) and do not accept further command calls after that.

## Entity model

`DocumentEntity` emits five event types: `DocumentUploaded`, `DocumentSanitized`, `DocumentChunked`, `DocumentIndexed`, `DocumentFailed`. `QueryEntity` emits four: `QuerySubmitted`, `RetrievalStarted`, `AnswerRecorded`, `QueryFailed`. Each view projects its entity's events into a read-model row. `DocumentSanitizer` and `ChunkIndexer` are Consumer components subscribing to `DocumentEntity` events; they act on those events without holding a reference to the entity state. `AnswerWorkflow` reads `QueryEntity` state (via command call) only in its `retrieveStep`; it also writes `QueryEntity` via `recordAnswer` and `fail` commands. `ChunkStore` is populated by `ChunkIndexer` and read by `AnswerWorkflow`.

## Defence-in-depth governance flow

For any answer that reaches a user, the source documents and the answer itself passed through:

1. **PII sanitizer** — the chunk index contains no raw PII; retrieved excerpts cannot leak identifiers.
2. **CitationAnswererAgent** — one model call, one structured output constrained to the received chunks.
3. **before-agent-response guardrail** — hallucinated chunkIds, empty excerpts, and out-of-enum confidence values are caught before the answer leaves the agent loop.

Each control is independent. Removing the sanitizer leaves PII in the index even if the guardrail is present. Removing the guardrail allows hallucinated citations even if the index is clean. Both must be present for the grounding guarantee to hold end-to-end.
