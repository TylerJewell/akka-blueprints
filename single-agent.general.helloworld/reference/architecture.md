# Architecture

Hello Agent is one autonomous agent behind an event-sourced question lifecycle. The four diagrams below are the source the Architecture tab renders. Each carries the Akka theme variables and, for the state machine, the Lesson 24 label-colour overrides.

## Component graph

See [`PLAN.md`](../PLAN.md#component-graph). A client posts a question to `AskEndpoint`, which writes an `ask` command to `QuestionEntity`. The resulting `QuestionAsked` event is consumed by `AnswerConsumer`, which runs the `QuestionAnswerer` agent's single ANSWER task and writes the result back to the entity. `QuestionsView` projects every entity event into the read model that `AskEndpoint` lists and streams over SSE. `AppEndpoint` serves the embedded UI.

The flow has no orchestration component: a single agent, a single write-back, no second hop. The only branch is at the write-back — the answer is either recorded or blocked.

## Interaction sequence

See [`PLAN.md`](../PLAN.md#interaction-sequence). The HTTP request returns the `questionId` immediately after the `ask` command persists; answering happens asynchronously off the `QuestionAsked` event. The `Note over` block marks where the before-agent-response check fires inside the agent. The `alt` branch shows the two write-back outcomes — `recordAnswer` when the check passes, `blockAnswer` when it fails.

## State machine

See [`PLAN.md`](../PLAN.md#state-machine). A question starts `ASKED`, then moves once to a terminal `ANSWERED` or `BLOCKED`. There is no re-ask transition; a new question is a new entity instance.

## Entity model

See [`PLAN.md`](../PLAN.md#entity-model). `QuestionEntity` is event-sourced from `QuestionAsked`, `AnswerRecorded`, and `AnswerBlocked`. The same `Question` record is both the entity state and the `QuestionsView` row type, so every nullable lifecycle field is `Optional<T>` (Lesson 6).
