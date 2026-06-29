# Architecture — post-incident-reviewer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `PIREndpoint` accepts a `{incidentId}` POST, writes `PIRCreated` onto `PIREntity`, and starts `PIRWorkflow` keyed by `"pir-" + pirId`. The workflow's first step (`gatherStep`) emits `GatherStarted`, then calls `PIRAgent` with `TaskDef.taskType(GATHER_EVIDENCE)` and a `phase = GATHER` metadata tag. The agent invokes `GatherTools.fetchIncidentRecord` and `GatherTools.fetchTimelineEvents`; once the agent returns an `EvidenceLog`, the workflow writes `EvidenceGathered` onto the entity and advances to `assessStep` — same pattern, the ASSESS task carries `phase = ASSESS`. Then `draftStep` runs with `phase = DRAFT`. After the DRAFT task responds, `PIRGuardrail` fires — checking structural quality before the response is committed — then `ReviewDrafted` lands, and `signoffStep` notifies the incident owner and pauses.

The graph has exactly one LLM-calling component. `PIRSignoffNotifier` is a deterministic side-effect class and does not call the model. That is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `gatherStep` and `assessStep`, the workflow writes `EvidenceGathered` onto the entity. `assessStep` then reads `evidenceLog` from the entity to build the ASSESS task's instruction context. The agent never sees gather-phase data inside the assess task's conversation; the typed handoff is the only path information travels.
2. The `before-agent-response` guardrail fires AFTER the DRAFT task returns but BEFORE `ReviewDrafted` is committed. A draft that fails any of the four quality checks is never written to the durable record; the agent retries with the structured rejection reason.
3. The HITL sign-off pause is durable. `signoffStep` persists its paused state in `PIREntity` (`AWAITING_SIGNOFF`). If the service restarts while the workflow is waiting, the workflow resumes from the same paused position. The incident owner's decision is the only path forward.

The agent calls are bounded by per-step timeouts (60 s on gather / assess, 90 s on draft). `signoffStep` has no timeout.

## State machine

Eleven states (CREATED, GATHERING, GATHERED, ASSESSING, ASSESSED, DRAFTING, DRAFTED, AWAITING_SIGNOFF, COMPLETE, REJECTED, FAILED). The interesting paths:

- The happy path walks `CREATED → GATHERING → GATHERED → ASSESSING → ASSESSED → DRAFTING → DRAFTED → AWAITING_SIGNOFF → COMPLETE`.
- A rejected sign-off lands in `REJECTED` from `AWAITING_SIGNOFF`.
- Three failure transitions land in `FAILED`: an agent error during `GATHERING`, `ASSESSING`, or `DRAFTING`. A `FAILED` review's prior data is preserved on the entity.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no auto-complete path. Every review that passes the guardrail must receive a human sign-off before reaching `COMPLETE`. This is the accountability invariant of the blueprint.

## Entity model

`PIREntity` is the source of truth. It emits thirteen event types — three lifecycle starts, three lifecycle completions (gather/assess/draft), sign-off request, sign-off record, final outcomes, the guardrail audit, and the failure. `PIRView` projects every event into a row used by the UI. `PIRWorkflow` both reads (`getPIR`) and writes on the entity across all steps. The relationship between `PIRAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any review that lands in the entity log, the incident walked through:

1. **Before-agent-response guardrail** — the draft is inspected once, after the DRAFT task completes. A draft missing an executive summary, action-item owners, valid timeline references, or a consistent severity classification is rejected before it becomes a durable event. A `GuardrailRejected` event records the violation for audit.
2. **PIRAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **HITL sign-off** — the incident owner reads the draft and approves or rejects it. The decision is a durable, auditable record on the entity. No review reaches `COMPLETE` without a human having read it.

Each layer is independent. The guardrail does not check whether the incident owner agrees; the sign-off does not re-run the quality checks. Removing one layer opens an explicit gap the other does not silently cover.
