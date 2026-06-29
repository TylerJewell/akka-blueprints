# Architecture — kb-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question and writes a `QuerySubmitted` event onto `QueryEntity`. The `PassageRetriever` Consumer subscribes, runs TF-IDF retrieval over the in-process `KbIndex`, and writes the resulting passages back via `attachPassages`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `KbAnswerAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and the retrieved passages (serialised JSON) as a `TaskDef.attachment(...)`. Once an answer returns, the workflow writes `AnswerRecorded` and runs `GroundednessScorer` in `evalStep`. The score lands as `GroundednessScored`. `KbView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

Separately, `QueryEndpoint` also accepts document ingestion requests (`POST /api/documents`). Each accepted document emits a `DocumentIndexed` event that `KbDocumentConsumer` subscribes to, tokenises, and upserts into `KbIndex`. This keeps the index live; a question asked immediately after a new document is indexed will retrieve passages from it.

The graph deliberately has no second agent. The on-decision evaluator (`GroundednessScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two moments where the system waits:

1. The `PassageRetriever` subscription lag between `QuerySubmitted` and `PassagesAttached` — sub-second in normal operation because retrieval is in-process.
2. The `awaitPassagesStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.passages().isPresent()` returns true.

The agent call itself is bounded by `answerStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The notable paths:

- The happy path is `SUBMITTED → RETRIEVING → ANSWERING → ANSWER_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a retriever error during `SUBMITTED`, and an agent error during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity; the UI shows the partial state for the operator.
- There is no `APPROVED` or `REJECTED` state. The answer is surfaced to the user with a groundedness score; trust decisions are left to the reader.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `KbView` projects every event into a row used by the UI. `PassageRetriever` subscribes to entity events to compute the retrieval result. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `recordGroundedness`, `fail`) on the entity. The relationship between `KbAnswerAgent` and `KbAnswer` is "returns" — the agent's task result is the answer record.

## Retrieval-augmented governance flow

For any answer that lands in the entity log, the question passed through:

1. **In-process TF-IDF retriever** — the model never invents sources; it only sees passages that actually exist in the indexed corpus.
2. **KbAnswerAgent** — one model call, one structured output with explicit citations.
3. **On-decision groundedness evaluator** — every answer gets a 1–5 score measuring how closely it traces back to the retrieved passages, surfaced immediately in the UI.

Each step is independent. The retriever can deliver zero passages (signalling "not in KB"), the agent can correctly declare NOT_FOUND, and the evaluator will score that correctly. There is no silent fallback to general-knowledge generation.
