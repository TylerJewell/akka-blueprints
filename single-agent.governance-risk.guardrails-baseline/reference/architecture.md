# Architecture — guardrails-baseline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call with two guardrail intercept points. `ModerationEndpoint` accepts a submission and writes a `MessageSubmitted` event onto `ModerationEntity`. The `MessageSanitizer` Consumer subscribes, strips PII, and writes the sanitized message back via `attachSanitized`. The same Consumer then starts a `ModerationWorkflow` instance. The workflow's `moderateStep` calls `ModerationAgent` — the single AutonomousAgent — with the policy rules as `TaskDef.instructions(...)` and the sanitized message as a `TaskDef.attachment(...)`.

Two guardrails are wired on the agent. `InputGuardrail` fires at the `before-agent-invocation` hook: it checks the agent's planned tool list against the policy's allowlist before any LLM call. `OutputGuardrail` fires at the `after-llm-response` hook: it validates each candidate `ModerationDecision` before it leaves the agent's loop. Once a decision passes both guardrails, the workflow writes `DecisionRecorded` and runs `AuditScorer` in `auditStep`. The score lands as `AuditCompleted`. `ModerationView` projects every entity event into a read-model row; `ModerationEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The audit scorer (`AuditScorer`) is a deterministic rule-based scorer — not an LLM. Exactly one component talks to a model, which is what makes this a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Two guardrail check points are explicit in the sequence:

1. After `runSingleTask`, the `InputGuardrail` inspects the agent's planned tools. On the happy path it accepts immediately; the LLM call proceeds.
2. After the LLM returns, `OutputGuardrail` validates the candidate decision structure. On the happy path it accepts; the decision proceeds to the workflow.

If either guardrail rejects, the agent loop retries within its 3-iteration budget. The sequence does not depict the retry path — see the J2 and J3 user journeys.

## State machine

Six states. The paths of interest:

- The happy path is `SUBMITTED → SANITIZED → MODERATING → DECISION_RECORDED → AUDITED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error or guardrail-exhaustion (all 3 iterations rejected) during `MODERATING`. A `FAILED` moderation's prior state is preserved on the entity — the UI shows the partial state for the operator.
- There is no `APPROVED` or `REJECTED` state. ALLOW/BLOCK/ESCALATE are properties of the decision record, not lifecycle states. An operator may act on an ESCALATE decision outside the system; the blueprint stops at `AUDITED`.

## Entity model

`ModerationEntity` is the source of truth. It emits six event types. `ModerationView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `ModerationWorkflow` both reads (`getModeration`) and writes (`markModerating`, `recordDecision`, `recordAudit`, `fail`) on the entity. The relationship between `ModerationAgent` and `ModerationDecision` is "returns" — the agent's task result is the decision record.

## Defence-in-depth governance flow

For any decision that lands in the entity log, the message passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **InputGuardrail** — the agent's tool access is checked against the policy allowlist before any LLM call.
3. **ModerationAgent** — one model call, one structured output.
4. **OutputGuardrail** — bad parses, missing rule results, out-of-enum actions, and verdict inconsistencies are caught before the response leaves the agent loop.
5. **AuditScorer** — every well-formed decision still gets a 1–5 consistency score so the operator knows which decisions to inspect.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
