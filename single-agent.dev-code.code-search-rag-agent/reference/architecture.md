# Architecture — code-search-rag-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a query submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `ChunkSanitizer` Consumer subscribes, redacts secrets from every retrieved code chunk, and writes the sanitized chunks back via `attachSanitizedChunks`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `CodeSearchAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and each sanitized chunk as a separate `TaskDef.attachment(...)`. Once the agent returns an answer, the workflow writes `AnswerRecorded` and runs `GroundingScorer` in `groundingStep`. The score lands as `GroundingScored`. `ChunkIndexView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The grounding scorer (`GroundingScorer`) is a deterministic rule-based component — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses in the flow:

1. The `ChunkSanitizer` subscription lag between `QuerySubmitted` and `ChunksSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.sanitized().isPresent()` returns true.

The agent call itself is bounded by `answerStep`'s 60 s timeout. When many chunks are attached the serialization overhead is small because each attachment is a plain text file; the LLM latency is the dominant term. The `groundingStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → CHUNKS_SANITIZED → ANSWERING → ANSWERED → GROUNDED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error or iteration-budget exhaustion during `ANSWERING`. A `FAILED` query's partial data is preserved on the entity — the UI shows what was completed before the failure.
- There is no `ACCEPTED` or `ACTED_ON` state. The answer is informational; the developer reads it and navigates to the cited file independently. The blueprint deliberately stops at `GROUNDED`.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `ChunkIndexView` projects every event into a row used by the UI. `ChunkSanitizer` subscribes to entity events to compute the sanitized chunk set. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `recordGrounding`, `fail`) on the entity. The relationship between `CodeSearchAgent` and `SearchAnswer` is "returns" — the agent's task result is the answer record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the code chunks passed through:

1. **Secret sanitizer** — the model never sees credentials or key material; the audit log retains the raw chunk content for developer review.
2. **CodeSearchAgent** — one model call, one structured output.
3. **On-answer grounding scorer** — every answer gets a 1–5 grounding score so the developer knows at a glance whether the cited file paths are real references in the retrieved set or hallucinated paths.

Each step is independent. Removing the sanitizer opens a direct secret-exposure path; removing the grounding scorer removes the signal that would tell the developer an answer is fabricated.
