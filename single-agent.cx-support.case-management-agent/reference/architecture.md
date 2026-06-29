# Architecture — case-management-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `CaseEndpoint` accepts an inbound message and writes a `MessageReceived` event onto `CaseEntity`. The `MessageSanitizer` Consumer subscribes, redacts PII, and writes the sanitized message back via `attachSanitized`. The same Consumer then starts a `CaseWorkflow` instance. The workflow's `agentStep` calls `SupportAgent` — the single AutonomousAgent — with the sanitized message and any open-case context as task instructions. Two guardrails are wired on the agent: `ActionGuardrail` fires on every candidate response via the `before-agent-response` hook; `ToolCallGuardrail` fires before each CRM tool invocation via the `before-tool-call` hook. Once a compliant action passes both guards, the workflow writes `ActionApplied` and runs `ActionEvaluator` in `evalStep`. The score lands as `EvaluationScored`. `CaseView` projects every entity event into a read-model row; `CaseEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-turn evaluator (`ActionEvaluator`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the two distinct pauses:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `CaseEntity` every 1 s up to its 15 s timeout, advancing as soon as `caseRecord.sanitizedMessage().isPresent()` returns true.

The agent call itself is bounded by `agentStep`'s 60 s timeout, which accommodates both LLM latency and potential guardrail-triggered retries. The `evalStep` is synchronous and finishes in milliseconds.

Two additional interactions happen inside the agent turn that the sequence shows explicitly:

- `ActionGuardrail` fires on each candidate response. A reject sends the agent back for another iteration.
- `ToolCallGuardrail` fires before the CRM tool executes. A policy violation returns a reason string; the agent may revise the tool arguments and retry.

## State machine

Six states. Key paths:

- The happy path is `OPEN → IN_PROGRESS → RESOLVED` (create then close) or `OPEN → IN_PROGRESS → ESCALATED → IN_PROGRESS → RESOLVED` (create, escalate, work, close).
- `ESCALATED → IN_PROGRESS` happens when a higher-tier team picks up the case and begins work, which produces an UPDATE action.
- Two terminal failure transitions land in `FAILED`: a sanitizer error during `OPEN`, and an agent error (or guardrail-exhaustion) during any active state.
- There is no auto-close. The `RESOLVED` terminal state is only reached when the agent produces a `CLOSE` action — deliberate, so a premature close is the agent's explicit decision rather than a timeout.

## Entity model

`CaseEntity` is the source of truth. It emits six event types. `CaseView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `CaseWorkflow` both reads (`getCase`) and writes (`markActing`, `applyAction`, `recordEvaluation`, `fail`) on the entity. The relationship between `SupportAgent` and `CaseAction` is "returns" — the agent's task result is the action record, which the workflow then applies.

## Defence-in-depth governance flow

For any action that lands in the entity log, the inbound message passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **SupportAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — missing fields, out-of-enum values, and missing target-case ids are caught before the response leaves the agent loop.
4. **before-tool-call guardrail** — escalation-path violations, writes to non-existent cases, and blank required fields are caught before they reach the CRM store.

Each step is independent. Removing any one of them opens a distinct gap the others do not silently cover.
