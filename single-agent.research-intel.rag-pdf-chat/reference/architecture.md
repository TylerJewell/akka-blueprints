# Architecture — rag-pdf-chat

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ChatEndpoint` accepts a PDF upload and writes a `DocumentUploaded` event onto `PdfDocumentEntity`. The `PassageRetriever` Consumer subscribes, chunks the PDF into fixed-size passages, computes term-frequency vectors in process, and writes the indexed passage list back via `attachIndex`. On each user question, `ChatEndpoint` starts a `ChatSessionWorkflow` instance. The workflow's `retrieveStep` calls `PassageRetriever.rankPassages(documentId, questionText, 5)` directly — no LLM call, no external service. The top-five passages are passed as a `TaskDef.attachment(...)` to `PdfChatAgent`. The agent's `before-agent-response` guardrail (`CitationGuardrail`) validates each candidate response against the supplied passage ids. Once a valid `CitedAnswer` passes, the workflow writes `AnswerRecorded` to `ChatSessionView`; `ChatEndpoint` serves the view over REST and SSE.

The graph has no second agent. `PassageRetriever` is a deterministic Consumer and direct-call component that talks to no model. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1), split into two user actions: upload and ask.

**Upload path:** `ChatEndpoint` writes `DocumentUploaded` to `PdfDocumentEntity`; the `PassageRetriever` Consumer picks it up, runs the chunking and indexing pipeline, and writes `DocumentIndexed` back. The whole round trip is sub-second in normal operation.

**Ask path:** `ChatEndpoint` starts `ChatSessionWorkflow`. The `retrieveStep` calls `PassageRetriever.rankPassages` — a synchronous in-process call bounded by the 10 s step timeout. The returned passages are attached to the agent task in `answerStep`, which is bounded by the 60 s step timeout. The `CitationGuardrail` runs on every agent iteration before any answer leaves the loop.

## Document state machine

Three states. The upload path is `UPLOADING → INDEXED`. If the indexer encounters an unreadable payload it transitions to `FAILED`. A `FAILED` document's upload payload is preserved on the entity — the UI shows the document title and error reason.

## Exchange state machine

Five terminal-reachable states. The happy paths are `RETRIEVING → ANSWERING → ANSWERED` (for answerable questions) and `RETRIEVING → ANSWERING → UNANSWERABLE` (for questions the document cannot answer). Two failure transitions land in `FAILED`: a retriever error during `RETRIEVING`, and an agent error or guardrail-exhaustion during `ANSWERING`. The `UNANSWERABLE` state is a normal outcome, not a failure — the agent explicitly returned `answerable = false` and the guardrail accepted the response.

## Entity model

`PdfDocumentEntity` is the source of truth for uploaded documents and their indexed passage lists. It emits three event types. `PassageRetriever` subscribes to entity events to index passages. `ChatSessionWorkflow` reads the entity (to get the documentId for passage retrieval) and writes answer-lifecycle events to `ChatSessionView`. The relationship between `PdfChatAgent` and `CitedAnswer` is "returns" — the agent's task result is the cited-answer record.

## Defence-in-depth governance flow

For any answer that lands in the session view, the question passed through:

1. **PassageRetriever** — only passages from the indexed document are supplied to the agent. The agent never has access to the full document corpus, only the top-K retrieved passages.
2. **PdfChatAgent** — one model call, one structured output, passages as an attachment.
3. **before-agent-response guardrail** — fabricated passage ids, missing citations on answerable responses, and non-conforming unanswerable responses are caught before the response leaves the agent loop.

The retriever boundary and the guardrail are independent. Removing the guardrail allows fabricated citations to reach the session log. Removing the passage-attachment pattern (and inlining the full document instead) removes the retrievability audit trail — the system would no longer know which passages the agent drew on.
