# Architecture — meeting-preparer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `BriefingEndpoint` accepts a meeting request and writes a `MeetingRequested` event onto `BriefingEntity`. The `ContactSanitizer` Consumer subscribes, redacts PII from the CRM snapshot, and writes the sanitized data back via `attachSanitizedData`. The same Consumer then starts a `BriefingWorkflow` instance. The workflow's `briefStep` calls `BriefingAgent` — the single AutonomousAgent — with the meeting context as `TaskDef.instructions(...)` and three attachments (`sanitized-crm.txt`, `financial-highlights.txt`, `news-items.txt`) as `TaskDef.attachment(...)` calls. Once a brief lands, the workflow writes `BriefReady` and runs `BriefingEvaluator` in `evalStep`. The eval score lands as `BriefEvaluated`. `BriefingView` projects every entity event into a read-model row; `BriefingEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`BriefingEvaluator`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct pauses:

1. The `ContactSanitizer` subscription lag between `MeetingRequested` and `ContactDataSanitized` — sub-second in normal operation, because the sanitizer runs a local regex pipeline with no network calls.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `BriefingEntity` every 1 s up to its 15 s timeout, advancing as soon as `briefing.sanitized().isPresent()` returns true.

The agent call itself is bounded by `briefStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `REQUESTED → DATA_SANITIZED → BRIEFING → BRIEF_READY → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `REQUESTED`, and an agent error during `BRIEFING`. A `FAILED` brief's partial data is preserved on the entity — the UI shows what arrived before the failure.
- There is no `APPROVED` state. The brief is advisory; the banker reads it and acts outside the system.

## Entity model

`BriefingEntity` is the source of truth. It emits six event types. `BriefingView` projects every event into a row used by the UI. `ContactSanitizer` subscribes to entity events to compute the sanitized CRM form. `BriefingWorkflow` both reads (`getBriefing`) and writes (`markBriefing`, `recordBrief`, `recordEval`, `fail`) on the entity. `BriefingAgent` returns `MeetingBrief` as its task result — the relationship is "returns", not "writes directly".

## Governance flow

For any brief that lands in the entity log, the CRM data passed through:

1. **PII sanitizer** — the model never sees contact names, emails, or phone numbers; the audit log retains the raw CRM snapshot.
2. **BriefingAgent** — one model call, one structured output (MeetingBrief).
3. **On-decision evaluator** — every brief gets a 1–5 evidence score so the banker knows at a glance which briefs are well-grounded and which warrant a closer look before the call.

Each step is independent. Removing the sanitizer leaves identifiers in the model's input. Removing the evaluator leaves thin briefs invisible.
