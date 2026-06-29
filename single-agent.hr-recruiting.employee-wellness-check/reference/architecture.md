# Architecture — wellness-check-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `CheckInEndpoint` accepts a campaign schedule and individual check-in submissions. When a check-in arrives, `CheckInEntity` emits `ResponseReceived`. The `ResponseSanitizer` Consumer subscribes, redacts special-category wellness markers, and writes the sanitized response back via `attachSanitized`. The same Consumer starts a `CheckInWorkflow` instance. The workflow's `analyseStep` calls `WellnessCheckAgent` — the single AutonomousAgent — with the check-in questions as `TaskDef.instructions(...)` and the sanitized response as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ResponseGuardrail`) validates each candidate response and, if `crisisFlag == true`, emits `CrisisEscalated` on `CheckInEntity` before accepting. Once a valid analysis passes, the workflow writes `AnalysisRecorded` and runs `MoraleSurveillance` in `surveillanceStep`. The campaign-level risk signal lands as `SurveillanceEvaluated`. `CheckInView` and `CampaignView` project entity events into read-model rows; `CheckInEndpoint` serves both over REST and SSE.

The graph has no second agent. `MoraleSurveillance` is a deterministic rule-based evaluator — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system pauses:

1. The `ResponseSanitizer` subscription lag between `ResponseReceived` and `ResponseSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `CheckInEntity` every 1 s up to its 15 s timeout, advancing as soon as `checkIn.sanitized().isPresent()` returns true.

The agent call itself is bounded by `analyseStep`'s 60 s timeout. The `surveillanceStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Seven states for `CheckInEntity`. The significant paths:

- The happy path is `RECEIVED → SANITIZED → ANALYSING → ANALYSIS_RECORDED → EVALUATED`.
- When `crisisFlag == true`, the path diverges at `ANALYSIS_RECORDED → ESCALATED`. The check-in is terminal at `ESCALATED`; it does not proceed to `EVALUATED`. It is excluded from normal morale aggregation.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (or guardrail-exhaustion) during `ANALYSING`. A `FAILED` check-in's prior data is preserved on the entity.
- There is no `APPROVED` or `ACTIONED` state. The analysis is advisory; the HR coordinator reads it and acts outside the system. The blueprint stops at `EVALUATED` or `ESCALATED`.

## Entity model

`CheckInEntity` and `CampaignEntity` are the two sources of truth. `CheckInEntity` emits seven event types. `CampaignEntity` emits four. `CheckInView` projects check-in events into a row for the UI. `CampaignView` projects both campaign events and check-in `AnalysisRecorded` events — that dual subscription is what lets the campaign aggregate counts stay current without a join query. `ResponseSanitizer` subscribes to check-in events. `CheckInWorkflow` both reads (`getCheckIn`) and writes (`markAnalysing`, `recordAnalysis`, `escalateCrisis`, `recordSurveillance`, `fail`) on the entity. The relationship between `WellnessCheckAgent` and `CheckInAnalysis` is "returns" — the agent's task result is the analysis record.

## Defence-in-depth governance flow

For any analysis that lands in the entity log, the check-in response passed through:

1. **Special-category sanitizer** — the model never sees wellness markers; the audit log retains the raw form.
2. **WellnessCheckAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, out-of-enum morale levels, and missing recommendations are caught before the response leaves the agent loop. Crisis signals are intercepted here and routed to a human at the earliest possible moment.
4. **Post-market-surveillance evaluator** — every campaign accumulates a morale-risk score so an HR decision-maker knows which campaigns warrant a closer look before any workforce action is taken.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
