# Architecture — notion-rag

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question and writes a `QuestionSubmitted` event onto `SessionEntity`. The `NotionRetriever` Consumer subscribes, calls the Notion database query API, and writes the retrieved rows back via `attachRows`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `NotionQueryAgent` — the single AutonomousAgent — with the user's question as `TaskDef.instructions(...)` and the retrieved rows JSON as a `TaskDef.attachment("rows.json", ...)`. The agent's `before-agent-response` guardrail (`GroundingGuardrail`) validates each candidate response. Once an answer passes, the workflow writes `AnswerRecorded` to `SessionEntity`. `SessionView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `NotionRetriever` is a Consumer that calls an HTTP API — it makes no model call. `GroundingGuardrail` is deterministic logic — it makes no model call. Exactly one component talks to a model, which is what makes this a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses exist in the flow:

1. The `NotionRetriever` subscription lag between `QuestionSubmitted` and `RowsRetrieved` — depends on Notion API round-trip time; sub-second in mock mode.
2. The `awaitRowsStep` polling loop inside the workflow — polls `SessionEntity` every 1 s up to its 15 s timeout, advancing as soon as the question's status equals `ROWS_RETRIEVED`.

The agent call is bounded by `answerStep`'s 60 s timeout to accommodate model latency.

## State machine

Five question states. The interesting paths:

- The happy path is `SUBMITTED → ROWS_RETRIEVED → ANSWERING → ANSWERED`.
- Two failure transitions land in `FAILED`: a retriever error during `SUBMITTED`, and an agent error (or guardrail exhaustion) during `ANSWERING`. A `FAILED` question preserves whatever data landed before the failure — the session history still shows the question text and, if rows were retrieved, the row count.
- There is no `APPROVED` or `REJECTED` state. Answers are informational; the user reads them and acts outside the system.

## Entity model

`SessionEntity` is the source of truth. It holds all questions for a session as a list inside the entity state. It emits six event types. `SessionView` projects every event into a row used by the UI. `NotionRetriever` subscribes to entity events to trigger retrieval. `QueryWorkflow` reads (`getSession`) and writes (`markAnswering`, `recordAnswer`, `failQuestion`) on the entity. `NotionQueryAgent` returns `QueryAnswer` — the agent's task result is the answer record.

## Grounding governance flow

For any answer that lands in the entity log, the content passed through:

1. **NotionRetriever** — only rows actually returned by the Notion API reach the agent. The retrieval boundary is enforced at the Consumer, not by the model.
2. **NotionQueryAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — ungrounded row citations and empty citation lists are caught before the response leaves the agent loop, forcing a retry.

Each step is independent. Removing the grounding check opens an explicit gap — the retriever would still run but the model could freely hallucinate row content.
