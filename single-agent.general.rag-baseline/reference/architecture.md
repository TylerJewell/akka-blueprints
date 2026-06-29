# Architecture — rag-baseline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `PassageRetriever` Consumer subscribes, calls the in-process `VectorStore` with the question text, and writes the top-K passages back via `attachPassages`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `AnswerAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and each retrieved passage as a separate `TaskDef.attachment(...)`. Once the agent returns a `GroundedAnswer`, the workflow writes `AnswerRecorded` and runs `GroundednessEvaluator` in `evalStep`. The score lands as `EvaluationScored`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The groundedness evaluator (`GroundednessEvaluator`) is a deterministic rule-based scorer that checks citation membership and completeness — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system waits:

1. The `PassageRetriever` subscription lag between `QuerySubmitted` and `PassagesRetrieved` — sub-second in normal operation because the `VectorStore` search is in-process.
2. The `awaitPassagesStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.retrieved().isPresent()` returns true.

The agent call itself is bounded by `answerStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The notable paths:

- The happy path is `SUBMITTED → PASSAGES_RETRIEVED → ANSWERING → ANSWER_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a retriever error during `SUBMITTED`, and an agent error (or iteration-budget exhaustion) during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state for the user.
- There is no `ACCEPTED` or `REJECTED` state. The answer is informational; the caller reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `QueryView` projects every event into a row used by the UI. `PassageRetriever` subscribes to entity events to trigger retrieval and then workflow creation. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `recordEvaluation`, `fail`) on the entity. The relationship between `AnswerAgent` and `GroundedAnswer` is "returns" — the agent's task result is the answer record.

## Groundedness evaluation flow

For any answer that lands in the entity log, the evidence chain is:

1. **VectorStore** — selects the candidate passages via cosine similarity; the model never sees passages that were not retrieved.
2. **AnswerAgent** — one model call, one structured output citing retrieved passage ids.
3. **GroundednessEvaluator** — checks that every cited id is genuine and the citation count matches; scores 1–5 and surfaces on the UI card.

The evaluator does not block the answer. A score of 1 or 2 is a signal to the caller to treat the answer with caution — it does not prevent it from being recorded or displayed. This design keeps the informational-answer contract honest: the system always produces an answer, and it always tells you how much to trust it.
