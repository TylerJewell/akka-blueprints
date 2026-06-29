# Architecture — service-agent-omnichannel

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one decision-making LLM call. `CaseEndpoint` accepts an inbound message and writes a `MessageReceived` event onto `CaseEntity`. The `MessageSanitizer` Consumer subscribes, redacts PII, and writes the sanitized message back via `attachSanitized`. The same Consumer then starts a `CaseWorkflow` instance. The workflow's `triageStep` classifies the message (using `CaseTriage`, no LLM) and records the `CaseCategory`. The `handleStep` calls `ServiceAgent` — the single AutonomousAgent — with the case context as `TaskDef.instructions(...)` and the sanitized message as a `TaskDef.attachment(...)`.

Two guardrails are registered on the agent: `ReplyGuardrail` fires on every `before-agent-response` candidate, and `CrmWriteGuardrail` fires on every `before-tool-call` attempt. Both run inside the agent loop before any reply or tool result propagates outward. Once a reply passes both checks, the workflow writes `ReplySent`. If the agent chose `ESCALATE`, it also calls `CaseEntity.escalate(reason)`. Then `EscalationScorer` runs in `evalStep`. The score lands as `EvaluationScored`. `CaseView` projects every entity event into a read-model row; `CaseEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one component that talks to a model (`ServiceAgent`). Triage (`CaseTriage`), escalation scoring (`EscalationScorer`), and both guardrails are deterministic or rule-based — they have no LLM calls of their own.

## Interaction sequence

The sequence traces the happy path (J1): a billing-dispute message arrives on the WEB channel, the agent creates a CRM case, and the reply lands in `REPLIED` state.

Two distinct moments where the system waits:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `CaseEntity` every 1 s up to its 15 s timeout, advancing as soon as `case.sanitized().isPresent()` returns true.

The agent call is bounded by `handleStep`'s 60 s timeout. Both guardrails run synchronously inside the agent loop; they do not add async round-trips. The `triageStep` and `evalStep` are synchronous and finish in milliseconds.

## State machine

Eight states. The principal paths:

- **Happy path (non-escalation):** `RECEIVED → SANITIZED → TRIAGING → HANDLING → REPLIED → RESOLVED`.
- **Escalation path:** `RECEIVED → SANITIZED → TRIAGING → HANDLING → REPLIED → ESCALATED`. The `ESCALATED` state is terminal inside the system; a human operator handles the rest.
- **Failure:** two failure transitions land in `FAILED` — a sanitizer error during `RECEIVED`, and an agent error (or guardrail-exhaustion after 3 iterations) during `HANDLING`. A `FAILED` case retains all prior data on the entity.
- `REPLIED` is a brief intermediate state: if `resolutionIntent == ESCALATE` the workflow calls `CaseEntity.escalate` immediately after recording the reply, so REPLIED→ESCALATED can appear near-simultaneous in the SSE stream.

## Entity model

`CaseEntity` is the source of truth. It emits nine event types. `CaseView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `CaseWorkflow` both reads (`getCase`) and writes (`attachSanitized`, `recordTriage`, `markHandling`, `recordReply`, `escalate`, `resolve`, `recordEvaluation`, `fail`) on the entity. The relationship between `ServiceAgent` and `AgentReply` is "returns" — the agent's task result is the reply record.

## Defence-in-depth governance flow

For any reply that lands in the entity log, the customer message passed through:

1. **PII sanitizer** — the model never sees customer identifiers; the audit log retains the raw form.
2. **ServiceAgent** — one model call, one structured output.
3. **before-tool-call guardrail** — invalid CRM writes are blocked before they execute.
4. **before-agent-response guardrail** — channel-format violations, prohibited phrases, and structural errors are caught before the reply leaves the agent loop.
5. **On-decision evaluator** — every reply still gets a 1–5 appropriateness score so operators know which cases to spot-check.
6. **HITL escalation** — cases requiring human judgment are explicitly surfaced in a human queue and receive no further autonomous handling.

Each layer is independent. Removing any one of them opens an explicit gap the others do not silently cover.
