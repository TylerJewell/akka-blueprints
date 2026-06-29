# Architecture — hr-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a query submission and writes a `QuerySubmitted` event onto `QueryEntity`. The `QuerySanitizer` Consumer subscribes, runs two sequential redaction passes (PII then special-category), and writes the sanitized context back via `attachSanitized`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `answerStep` calls `HrPolicyAgent` — the single AutonomousAgent — with the sanitized query text as `TaskDef.instructions(...)` and the assembled employee profile plus policy corpus excerpt as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`PolicyAnswerGuardrail`) validates each candidate response. Once an answer passes, the workflow writes `AnswerRecorded`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The `QuerySanitizer` logic is deterministic code — two regex-and-heuristic passes — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system waits:

1. The `QuerySanitizer` subscription lag between `QuerySubmitted` and `QuerySanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.sanitized().isPresent()` returns true.

The agent call itself is bounded by `answerStep`'s 60 s timeout. Within the sanitizer, the two passes are sequential and synchronous — `QuerySanitized` is emitted only after both passes complete on the combined `SanitizedQueryContext`.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → ANSWERING → ANSWER_RECORDED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ANSWERING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state so an HR professional can inspect what the agent received.
- There is no `APPROVED` state. The answer is informational; the employee reads it and decides whether to act or escalate to HR. The blueprint deliberately stops at `ANSWER_RECORDED`.

## Entity model

`QueryEntity` is the source of truth. It emits five event types. `QueryView` projects every event into a row used by the UI. `QuerySanitizer` subscribes to entity events to compute the sanitized context. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnswering`, `recordAnswer`, `fail`) on the entity. The relationship between `HrPolicyAgent` and `PolicyAnswer` is "returns" — the agent's task result is the answer record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the query passed through:

1. **PII sanitizer (S1)** — the model never sees employee identifiers, emails, or phone numbers; the audit log retains the raw query.
2. **Special-category sanitizer (S2)** — health conditions, disability status, religious-accommodation phrases, union-membership indicators, and pregnancy-status disclosures are stripped from both the query text and the resolved employee profile before the model sees either.
3. **HrPolicyAgent** — one model call, one structured `PolicyAnswer` with citations.
4. **before-agent-response guardrail** — empty answers, out-of-enum verdicts, and citations referencing non-existent policy ids are caught before the response leaves the agent loop.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover — the special-category pass in particular cannot be replaced by the PII pass because special-category phrases do not follow regex-detectable formats.
