# Architecture — bm25-rag-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `BM25Retriever` Consumer subscribes, runs an Okapi BM25 search over the in-process `CorpusIndex` (loaded from `passages.jsonl` at startup), and writes the top-K passage references back via `attachPassages`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `CorpusQueryAgent` — the single AutonomousAgent — with the question text as `TaskDef.instructions(...)` and each retrieved passage body as a separate `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`CitationGuardrail`) validates each candidate response. Once an answer passes, the workflow writes `AnswerRecorded` and runs `GroundingScorer` in `scoringStep`. The grounding result lands as `GroundingScored`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The grounding scorer (`GroundingScorer`) is a deterministic rule-based component — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where the system waits:

1. The `BM25Retriever` subscription lag between `QuerySubmitted` and `PassagesAttached` — sub-second because BM25 is in-process.
2. The `awaitPassagesStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.passages().isPresent()` returns true.

The agent call is bounded by `answerStep`'s 60 s timeout. The `scoringStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. Key paths:

- The happy path is `SUBMITTED → PASSAGES_ATTACHED → ANSWERING → ANSWER_RECORDED → SCORED`.
- Two failure transitions land in `FAILED`: a retriever error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state so the user knows what was retrieved before the failure.
- There is no `ACCEPTED` or `REJECTED` state. The answer is informational; the user reads it and decides what to do with it. The blueprint deliberately stops at `SCORED`.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `QueryView` projects every event into a read-model row used by the UI. `BM25Retriever` subscribes to entity events to run retrieval and attach results. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `recordGrounding`, `fail`) on the entity. The relationship between `CorpusQueryAgent` and `QueryAnswer` is "returns" — the agent's task result is the answer record.

## Retrieval-then-generate governance flow

For any answer that lands in the entity log, the question passed through:

1. **BM25 retrieval** — the model is never called without a grounded evidence set; the retrieved passages are the model's only allowed sources.
2. **CorpusQueryAgent** — one model call, one structured output with explicit citation ids.
3. **before-agent-response guardrail** — citation ids not in the delivered attachment set and invalid answerType values are caught before the response leaves the agent loop.
4. **Grounding scorer** — every well-formed answer still gets a 1–5 score checking verbatim-quotation fidelity, so the user knows which answers are closely grounded versus loosely paraphrased.

Each step is independent. Removing the guardrail allows hallucinated passage ids to reach the entity log; removing the grounding scorer removes the observability signal on answer quality.
