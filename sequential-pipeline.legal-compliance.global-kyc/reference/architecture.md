# Architecture — global-kyc-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `KycEndpoint` accepts a `{applicantId, jurisdiction, documentTypes}` POST, writes `CaseCreated` onto `KycCaseEntity`, and starts `KycPipelineWorkflow` keyed by `"kyc-" + caseId`. The workflow's first step (`collectStep`) emits `CollectStarted`, then calls `KycAgent` with `TaskDef.taskType(COLLECT_DOCUMENTS)` and a `phase = COLLECT` metadata tag. The agent invokes `CollectTools.fetchDocument` and `CollectTools.lookupJurisdictionRules`; both reads come from the in-process sample-data corpus. Once the agent returns a `DocumentSet`, the workflow writes `DocumentsCollected` onto the entity. Before that event reaches `KycView`, `PiiSanitizer` intercepts the projection write and replaces the three sensitive fields with stable tokens, emitting `SanitizerApplied` for audit. The workflow then advances to `verifyStep` — same pattern, the VERIFY task carries `phase = VERIFY`. Then `decideStep` runs with `phase = DECIDE`. After the agent returns a `KycDecision`, the workflow inspects the outcome: a PASS routes directly to `evalStep`; a DECLINE, REFER, or PENDING_DOCUMENTS routes to `reviewStep`, which pauses until a compliance officer posts a resolution. After the HITL gate resolves (or is bypassed on PASS), `evalStep` runs `DecisionScorer` over the recorded `(KycDecision, VerificationResult, DocumentSet)` triple — no LLM call — and writes `DecisionEvaluated`. `KycView` projects every event into a read-model row (with PII already tokenised); `KycEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PiiSanitizer`, `HitlReviewGate`, and `DecisionScorer` are deterministic or operator-driven; that is what makes this a faithful **single-agent sequential-pipeline** example with three independent governance mechanisms.

## Interaction sequence

The sequence traces the happy-path PASS case (J1). Three properties are worth noting:

1. The task boundary IS the dependency contract. Between `collectStep` and `verifyStep`, the workflow writes `DocumentsCollected` onto the entity. The next step then reads `documents` from the entity to build the VERIFY task's instruction context. The agent never sees collect-phase context inside the verify task's conversation; the typed handoff is the only path information travels between phases.
2. PII containment happens on the view-projection path, not inside the agent call. This placement means the raw `DocumentSet` is locked to the entity event log regardless of what any downstream change does to the agent or the workflow; the sanitizer fires structurally, before every write to the read model.
3. The HITL gate is a workflow-level pause, not an agent retry. The agent finishes its DECIDE task and returns a typed result. The workflow then decides — based on the `outcome` field — whether to route through review. No additional LLM call is involved in the review step.

## State machine

Eleven states. The interesting paths:

- The happy PASS path walks `CREATED → COLLECTING → COLLECTED → VERIFYING → VERIFIED → DECIDING → DECIDED → EVALUATED`.
- The DECLINE path diverges at `DECIDING → PENDING_REVIEW → REVIEWED → DECIDED → EVALUATED`. A compliance officer must explicitly POST `/api/cases/{id}/review` before `REVIEWED` is reached.
- Three failure transitions land in `FAILED`: an agent error during COLLECTING, VERIFYING, or DECIDING (after retry exhaustion). A FAILED case's partial data is preserved on the entity — the UI shows the partial state.
- `SanitizerApplied` is a side-event recorded for audit; it does not transition status.

There is no `CLOSED` or `ARCHIVED` state. The pipeline stops at `EVALUATED`; case archival and retention are deployer responsibilities.

## Entity model

`KycCaseEntity` is the source of truth. It emits twelve event types — four lifecycle starts (`CollectStarted`, `VerifyStarted`, `DecideStarted`, `ReviewRequested`), four lifecycle completions (`DocumentsCollected`, `IdentityVerified`, `DecisionRendered`, `ReviewResolved`), the evaluation, the PII audit event, the failure, and the initial creation. `KycView` projects every event into a row used by the UI, with PII fields tokenised by `PiiSanitizer`. `KycPipelineWorkflow` both reads (`getCase`) and writes (twelve distinct commands) on the entity. The relationship between `KycAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any case that lands in the entity log, the applicant data passed through three independent governance layers:

1. **PII sanitizer** — every write to the view carries only tokenised identity fields. The entity event log retains raw values; every other consumer (view, SSE, UI, any downstream export) sees tokens. The `SanitizerApplied` event records exactly which fields were redacted.
2. **HITL application gate** — every non-PASS outcome is held at `PENDING_REVIEW` until a named compliance officer posts an explicit resolution. The officer's identity and decision are event-logged alongside the agent's reasoning.
3. **On-decision evaluator** — every `KycDecision` gets a 1–5 completeness-and-citation score. Rule citation coverage, rule ID validity, document verification, and outcome validity are each worth one point. Score ≤ 2 flags the case for manual inspection.

Each layer is independent. The sanitizer does not check decision quality; the HITL gate does not check PII containment; the evaluator does not check whether PII was sanitised. Removing any one of them opens an explicit gap that the other two do not silently cover.
