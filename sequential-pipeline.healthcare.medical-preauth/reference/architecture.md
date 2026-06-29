# Architecture — medical-preauth

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `PreAuthEndpoint` accepts a submission POST, writes `RequestSubmitted` onto `PreAuthEntity`, and starts `PreAuthWorkflow` keyed by `"preauth-" + requestId`. The workflow's first step (`validateStep`) emits `ValidationStarted`, then calls `PreAuthAgent` with `TaskDef.taskType(VALIDATE_REQUEST)` and a `phase = VALIDATE` metadata tag. The agent invokes `ValidateTools.checkMemberEligibility` and `ValidateTools.validateProcedureCode`; every call passes through `PhiSanitizer` first, which strips PHI fields before the payload reaches the LLM. Once the agent returns a `ValidationResult`, the workflow writes `RequestValidated` onto the entity and advances to `reviewStep` — same pattern, the REVIEW task carries `phase = REVIEW`. Then `determineStep` runs with `phase = DETERMINE`. After `DeterminationWritten` lands, `evalStep` runs `ClinicalCompletenessScorer` over the recorded triple — no LLM call — and writes `EvaluationScored`. If the determination outcome is `DENIED`, `humanHoldStep` fires next and the workflow waits for a reviewer acknowledgement. `PreAuthView` projects every event into a read-model row; `PreAuthEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `ClinicalCompletenessScorer` is a deterministic rule-based scorer and `PhiSanitizer` is a deterministic string processor; neither makes an LLM call. That is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1) for an `APPROVED` determination. Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `validateStep` and `reviewStep`, the workflow writes `RequestValidated` onto the entity. The next step then reads `validationResult` from the entity to build the REVIEW task's instruction context. The agent never sees validate-phase context inside the review task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `PhiSanitizer`. The sanitizer scans the payload for PHI fields before forwarding it, replaces each identified field with a request-scoped hash, and records a `SanitizationApplied` event. The phase-gate portion of the same hook also rejects misordered tool calls. A misordered call is rejected before the tool body executes; a PHI-bearing payload is stripped before the LLM receives it.

The agent calls themselves are bounded by per-step timeouts (60 s on validate / review / determine). `evalStep` is synchronous and finishes in milliseconds. `humanHoldStep` has a 72-hour timeout, matching the payer's operational SLA for denial review.

## State machine

Twelve states. The interesting paths:

- The happy path for an approved request walks `SUBMITTED → VALIDATING → VALIDATED → REVIEWING → POLICY_REVIEWED → DETERMINING → DETERMINED → EVALUATED`.
- A denied request branches at `DETERMINED → PENDING_HUMAN_REVIEW`. A reviewer acknowledgement drives `PENDING_HUMAN_REVIEW → EVALUATED`. A 72-hour timeout drives `PENDING_HUMAN_REVIEW → ESCALATED`.
- Three failure transitions land in `FAILED`: an agent error during `VALIDATING`, `REVIEWING`, or `DETERMINING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state.
- `SanitizationApplied` and `PhaseGuardrailRejected` are side-events recorded for audit; they do not transition status.

There is no `PUBLISHED` or `TRANSMITTED` state. The determination is the terminal output. Delivery to the requesting clinician's EHR is outside the scope of this blueprint.

## Entity model

`PreAuthEntity` is the source of truth. It emits fourteen event types — three lifecycle starts, three lifecycle completions, the evaluation, the human-hold pair, the escalation, the two audit side-events (sanitization and phase-gate rejection), the failure, and the initial submission. `PreAuthView` projects every event into a row used by the UI. `PreAuthWorkflow` both reads and writes on the entity across all five steps. The relationship between `PreAuthAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any determination that lands in the entity log, the submitted request passed through:

1. **PHI sanitizer** — every tool call payload is scanned for PHI fields before the LLM receives it. A `SanitizationApplied` event records each field replaced. If the payload cannot be parsed, the call is blocked entirely.
2. **Phase-gate** (second responsibility of `PhiSanitizer`) — every tool call is checked against the current entity status. A DETERMINE-phase tool called during REVIEWING is rejected before the tool body runs; a `PhaseGuardrailRejected` event records the violation for audit.
3. **PreAuthAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
4. **On-decision evaluator** — every determination gets a 1–5 clinical completeness score. Procedure code coverage, policy citation, rationale adequacy, and outcome validity are each worth one point on a base of 1.
5. **Human-in-the-loop hold** — every `DENIED` determination is held for human reviewer acknowledgement before it finalizes. The hold is durable: a service restart does not drop it.

Each layer is independent. The PHI sanitizer does not check policy coverage. The on-decision evaluator does not check PHI fields. Removing any one of them opens an explicit gap the others do not silently cover.
