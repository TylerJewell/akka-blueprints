# Architecture — conversational-interview

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call per interview turn. `SessionEndpoint` accepts a session-start request and writes a `SessionStarted` event onto `InterviewSessionEntity`. `InterviewWorkflow` fires immediately and invokes `InterviewConductorAgent` — the single AutonomousAgent — for the first question, passing the role definition and (initially empty) conversation history as `TaskDef.attachment(...)` calls. The agent's `before-agent-response` guardrail (`QuestionGuardrail`) validates each candidate question. Once a question passes, the workflow writes `TurnConducted` and exposes the question in the UI.

When the coordinator submits the candidate's answer, `SessionEndpoint` writes an `AnswerSubmitted` event. `AnswerSanitizer` Consumer subscribes, screens the answer for protected-category attributes, replaces flagged spans with typed redaction tokens, and writes `AnswerScreened` back via `attachScreenedAnswer`. The workflow resumes at `awaitScreenedStep`, polls until the screened answer is present, and then calls `InterviewConductorAgent` again with the updated history. This cycle repeats until the agent returns `sessionComplete: true`, at which point the workflow writes `SessionCompleted`.

`SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The `AnswerSanitizer` is a rule-based keyword screener — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1) through the first two turns. Two distinct pause points exist:

1. The `AnswerSanitizer` subscription lag between `AnswerSubmitted` and `AnswerScreened` — sub-second in normal operation.
2. The `awaitScreenedStep` polling loop inside the workflow — polls `InterviewSessionEntity` every 1 s up to its 15 s timeout, advancing as soon as the matched `ScreenedAnswer` is present in the history.

The agent call itself is bounded by `conductTurnStep`'s 60 s timeout. The workflow loops back to wait for the next answer after each non-final turn.

## State machine

Six states. The interesting paths:

- The happy path is `OPEN → CONDUCTING → ANSWER_RECEIVED → ANSWER_SCREENED → CONDUCTING → … → COMPLETED`. The `CONDUCTING → ANSWER_RECEIVED → ANSWER_SCREENED → CONDUCTING` cycle repeats once per interview turn.
- Two failure transitions land in `FAILED`: a startup error during `OPEN`, and an agent error or guardrail-exhaustion during `CONDUCTING`. A `FAILED` session's prior transcript is preserved on the entity.
- There is no `HIRED` or `REJECTED` state. The agent generates questions; all candidate-outcome decisions happen outside this system. The blueprint deliberately stops at `COMPLETED`.

## Entity model

`InterviewSessionEntity` is the source of truth. It emits six event types and holds the full turn list — each turn carrying the agent's question, the candidate's raw answer, and the screened answer. `SessionView` projects every event into a row used by the UI. `AnswerSanitizer` subscribes to entity events to compute the screened form. `InterviewWorkflow` both reads (`getSession`) and writes (`recordTurn`, `complete`, `fail`) on the entity. The relationship between `InterviewConductorAgent` and `InterviewTurn` is "returns" — the agent's task result is the next turn record.

## Defence-in-depth governance flow

For any question that reaches the coordinator's screen, the preceding answer passed through:

1. **Special-category sanitizer** — the model never sees protected-category attributes; the audit log retains the raw answer.
2. **InterviewConductorAgent** — one model call, one structured output (the next question).
3. **before-agent-response guardrail** — blank questions, off-competency mappings, prohibited topics, and leading language are caught before the response leaves the agent loop.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
