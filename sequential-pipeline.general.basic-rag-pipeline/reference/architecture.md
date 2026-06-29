# Architecture — basic-rag-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs two tasks in sequence. `RagEndpoint` accepts a `{question}` POST, writes `SessionCreated` onto `RagSessionEntity`, and starts `RagPipelineWorkflow` keyed by `"rag-" + sessionId`. The workflow's first step (`ingestStep`) emits `IngestStarted`, then calls `RagAgent` with `TaskDef.taskType(INGEST_CORPUS)` and a `phase = INGEST` metadata tag. The agent invokes `IngestTools.loadDocuments` then `IngestTools.indexChunks`, writing chunks into the in-process `VectorStore`. Once the agent returns an `IndexedCorpus`, the workflow writes `CorpusIndexed` onto the entity and advances to `queryStep`. The QUERY task's instruction context is built from the recorded `IndexedCorpus` plus the original question; the agent invokes `QueryTools.retrieveChunks` and `QueryTools.buildContext` to compose a grounded answer. Before the answer is returned to the workflow, `AnswerGuardrail` fires — inspecting every citation URL against the indexed corpus. On accept, the workflow writes `AnswerDrafted` and the guardrail records `GuardrailApplied{verdict: PASSED}`. `RagSessionView` projects every event into a read-model row; `RagEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `AnswerGuardrail` is a deterministic citation checker; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `ingestStep` and `queryStep`, the workflow writes `CorpusIndexed` onto the entity. The next step reads `corpus` from the entity to build the QUERY task's instruction context. The agent never sees ingest-phase context inside the query task's conversation; the typed handoff is the only path information travels.
2. The `before-agent-response` guardrail fires after the agent has composed its full answer but before that answer is committed to the workflow. A citation that traces to a URL not in the indexed corpus is caught here — before it reaches the caller — and the agent can self-correct within its iteration budget.

The agent calls themselves are bounded by per-step timeouts (60 s on ingest / query). The guardrail runs synchronously inside the agent's response path and adds negligible latency.

## State machine

Seven states. The interesting paths:

- The happy path walks `CREATED → INDEXING → INDEXED → ANSWERING → ANSWERED`.
- `BLOCKED` is reached when the agent's query iterations are exhausted without producing a grounded answer. The session's last recorded `GuardrailApplied` events carry the rejection reasons.
- Two failure transitions land in `FAILED`: an error during `INDEXING` or `ANSWERING` that exceeds the step recovery budget. A `FAILED` session's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailApplied{verdict: BLOCKED}` is recorded for each rejected draft answer; it does not change the status. Only budget exhaustion (`AnswerBlocked`) transitions to `BLOCKED`.

There is no `PUBLISHED` or `APPROVED` state. The answer is advisory; the user reads it and acts outside the system. The blueprint deliberately stops at `ANSWERED` or `BLOCKED`.

## Entity model

`RagSessionEntity` is the source of truth. It emits eight event types — two lifecycle starts (`IngestStarted`, `QueryStarted`), two lifecycle completions (`CorpusIndexed`, `AnswerDrafted`), the guardrail audit (`GuardrailApplied`), the budget-exhaustion terminal (`AnswerBlocked`), the creation event (`SessionCreated`), and the failure terminal (`SessionFailed`). `RagSessionView` projects every event into a row used by the UI. `RagPipelineWorkflow` both reads (`getSession`) and writes (`startIngest`, `recordCorpus`, `startQuery`, `recordAnswer`, `recordGuardrailOutcome`, `blockAnswer`, `fail`) on the entity. `RagAgent` returns two typed results — `IndexedCorpus` and `RagAnswer` — that become the payloads of `CorpusIndexed` and `AnswerDrafted` respectively.

## Governance flow

For any session that lands in the entity log, the question passed through:

1. **RagAgent — INGEST task**: documents loaded, chunks indexed into `VectorStore`. The typed `IndexedCorpus` is the dependency contract to the next phase.
2. **RagAgent — QUERY task**: relevant chunks retrieved via keyword similarity, answer composed. Tool calls are bounded to `retrieveChunks` and `buildContext`.
3. **Citation-grounding output guardrail**: every citation URL in the draft answer is checked against the recorded `IndexedCorpus.chunks[].sourceUrl`. Any unverifiable citation blocks the answer before it reaches the caller. The agent may retry within its 4-iteration budget; a budget-exhausted session surfaces `BLOCKED` rather than a fabricated answer.

Each step is independent. The ingest phase does not check citation quality; the guardrail does not re-run retrieval. Removing the guardrail opens a gap that the ingest phase does not cover.
